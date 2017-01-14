---
layout: post
title: "Conditionally serializing fields using Jackson"
date: 2015-04-17 19:11:57 +0300
comments: true
categories: [Java, Jackson]
---

When interacting with some REST API, we often deal with serialization of Java objects to JSON strings.
Lately, I came across a requirement to conditionally skip an object's field, according to its value.
Assume, for example, the following class:

``` Java
    public class MyType {
        private final double value;
    
        public MyType(final double value) {
            this.value = value;
        }
    
        @JsonProperty("value")
        public double getValue() {
            return value;
        }
    }
```

Assume we'd like to include ```value``` in the JSON serialized string only if its value is not equal to 0.
I'm using [Jackson](https://github.com/FasterXML/jackson) for JSON serialization, and the solution, as I found, was to implement and register a ```PropertyFilter```:

``` Java ProperyFilter interface http://fasterxml.github.io/jackson-databind/javadoc/2.5/com/fasterxml/jackson/databind/ser/PropertyFilter.html
public interface PropertyFilter {
    void serializeAsField(Object pojo, JsonGenerator jgen, SerializerProvider prov, PropertyWriter writer);

    void serializeAsElement(Object elementValue, JsonGenerator jgen, SerializerProvider prov, PropertyWriter writer) throws Exception;

    void depositSchemaProperty(PropertyWriter writer, JsonObjectFormatVisitor objectVisitor, SerializerProvider provider) throws JsonMappingException;

    @Deprecated 
    void depositSchemaProperty(PropertyWriter writer, ObjectNode propertiesNode, SerializerProvider provider) throws JsonMappingException;
}
```

In my case, only the first method required an special implementation:

``` Java ZeroValueProperyFilter
public class ZeroValueFilter implements PropertyFilter {
    void serializeAsField(Object pojo, JsonGenerator jgen, SerializerProvider prov, PropertyWriter writer) {
        if (pojo instanceof MyType && isValueFieldZero((MyType) pojo)) {
            return; // skip this field
        }
        writer.serializeAsField(pojo, jgen, prov);
    }

    private isValueFieldZero(MyType myClass) {
        return Double.compare(myClass.getValue(), 0.0) == 0;
    }

    void serializeAsElement(Object elementValue, JsonGenerator jgen, SerializerProvider prov, PropertyWriter writer) throws Exception {
        writer.serializeAsField(elementValue, jgen, prov);
    }

    void depositSchemaProperty(PropertyWriter writer, JsonObjectFormatVisitor objectVisitor, SerializerProvider provider) throws JsonMappingException {
        writer.depositSchemaProperty(objectVisitor);
    }

    @Deprecated 
    void depositSchemaProperty(PropertyWriter writer, ObjectNode propertiesNode, SerializerProvider provider) throws JsonMappingException {
        writer.depositSchemaProperty(propertiesNode, provider);
    }
}
```

I also had to annotate the class with the required filter:

``` Java MyType, annotated with filter
@JsonFilter("zeroValueFilter")
public class MyType {
    // as above
}
```

Lastly, upon creating an ```ObjectMapper``` to be used for JSON serialization, the filter should be registered:

``` Java ObjectMapper filter registration

final ObjectMapper mapper = new ObjectMapper();
final SimpleFilterProvider filterProvider = new SimpleFilterProvider()
        .addFilter("zeroValueFilter", new ZeroValueFilter());
mapper.setFilters(filterProvider);
```

We can now test the object mapper to see the filtering working:

``` Java
System.out.println(mapper.writeValueAsString(new MyType(42.0))); // prints {value:42.0}
System.out.println(mapper.writeValueAsString(new MyType(0.0)));  // prints {}
```

