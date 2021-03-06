# Manually implementing Deserialize for a struct

Only when [codegen](codegen.md) is not getting the job done.

The `Deserialize` impl below corresponds to the following struct:

```rust
struct Duration {
    secs: u64,
    nanos: u32,
}
```

Deserializing a struct is somewhat more complicated than [deserializing a
map](deserialize-map.md) in order to avoid allocating a String to hold the field
names. Instead there is a `Field` enum which is deserialized from a `&str`.

The implementation supports two possible ways that a struct may be represented
by a data format: as a seq like in Bincode, and as a map like in JSON.

```rust
impl Deserialize for Duration {
    fn deserialize<D>(deserializer: &mut D) -> Result<Self, D::Error>
        where D: Deserializer,
    {
        enum Field { Secs, Nanos };

        impl Deserialize for Field {
            fn deserialize<D>(deserializer: &mut D) -> Result<Field, D::Error>
                where D: Deserializer,
            {
                struct FieldVisitor;

                impl Visitor for FieldVisitor {
                    type Value = Field;

                    fn visit_str<E>(&mut self, value: &str) -> Result<Field, E>
                        where E: Error,
                    {
                        match value {
                            "secs" => Ok(Field::Secs),
                            "nanos" => Ok(Field::Nanos),
                            _ => Err(Error::unknown_field(value)),
                        }
                    }
                }

                deserializer.deserialize_struct_field(FieldVisitor)
            }
        }

        struct DurationVisitor;

        impl Visitor for DurationVisitor {
            type Value = Duration;

            fn visit_seq<V>(&mut self, mut visitor: V) -> Result<Duration, V::Error>
                where V: SeqVisitor,
            {
                let secs: u64 = match try!(visitor.visit()) {
                    Some(value) => value,
                    None => {
                        try!(visitor.end());
                        return Err(Error::invalid_length(0));
                    }
                };
                let nanos: u32 = match try!(visitor.visit()) {
                    Some(value) => value,
                    None => {
                        try!(visitor.end());
                        return Err(Error::invalid_length(1));
                    }
                };
                try!(visitor.end());
                Ok(Duration::new(secs, nanos))
            }

            fn visit_map<V>(&mut self, mut visitor: V) -> Result<Duration, V::Error>
                where V: MapVisitor,
            {
                let mut secs: Option<u64> = None;
                let mut nanos: Option<u32> = None;
                while let Some(key) = try!(visitor.visit_key::<Field>()) {
                    match key {
                        Field::Secs => {
                            if secs.is_some() {
                                return Err(<V::Error as Error>::duplicate_field("secs"));
                            }
                            secs = Some(try!(visitor.visit_value()));
                        }
                        Field::Nanos => {
                            if nanos.is_some() {
                                return Err(<V::Error as Error>::duplicate_field("nanos"));
                            }
                            nanos = Some(try!(visitor.visit_value()));
                        }
                    }
                }
                try!(visitor.end());
                let secs = match secs {
                    Some(secs) => secs,
                    None => try!(visitor.missing_field("secs")),
                };
                let nanos = match nanos {
                    Some(nanos) => nanos,
                    None => try!(visitor.missing_field("nanos")),
                };
                Ok(Duration::new(secs, nanos))
            }
        }

        const FIELDS: &'static [&'static str] = &["secs", "nanos"];
        deserializer.deserialize_struct("Duration", FIELDS, DurationVisitor)
    }
}
```
