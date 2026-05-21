# Issue: `DateTimeException` propagates unwrapped during Java `java.time` deserialisation

## Observed behaviour

When Jackson deserialises a `java.time` type (such as `LocalDate`, `ZonedDateTime`, or similar), it accepts syntactically valid JSON but may encounter a semantically invalid date — for example, month `13`, day `32`, or an out-of-range time component.

In these cases, the underlying Java `DateTimeException` is **not caught** by the deserialiser and propagates directly to the caller as an unchecked exception. This breaks the Jackson contract that all deserialisation failures should surface as `JsonMappingException` (or a subclass).

**Minimal reproduction:**

```java
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());

// "2024-13-01" is syntactically valid JSON but semantically invalid (month 13)
mapper.readValue("\"2024-13-01\"", LocalDate.class);
// Throws DateTimeException directly — should throw JsonMappingException
```

Applications that catch `JsonMappingException` to handle bad input will not catch this case, causing unexpected failures.

## Expected behaviour

Any `DateTimeException` thrown during deserialisation of a `java.time` type should be caught by the deserialiser and wrapped into a `JsonMappingException` (specifically `InvalidFormatException` or equivalent), with the original exception preserved as the cause.

The wrapping should:
- Apply consistently across all `java.time` type deserialisers (not just `LocalDate`).
- Preserve the original `DateTimeException` as the cause so stack traces remain useful.
- Not swallow exceptions that indicate programming errors rather than bad input.

## Notes

- The fix is needed across multiple individual type deserialisers in the `jackson-datatype-jsr310` module (now part of jackson-databind). Each deserialiser has its own parse/deserialise method that may throw `DateTimeException`.
- Look at the deserialiser hierarchy for Java time types to understand which classes need changes and whether there is a shared base class where some of the fix can be centralised.
- The wrapping strategy should be consistent — study how Jackson wraps other parsing exceptions (e.g., `NumberFormatException`) for guidance.
- There may be code paths in both string-based and numeric (epoch) deserialisation that need the same treatment.
- Tests should cover at least the most common types (`LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`) with invalid values.
