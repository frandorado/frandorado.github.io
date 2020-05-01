---
layout: post
title:  "Custom deserialization in Spring"
author: frandorado
categories: [spring]
tags: [spring, jackson, custom, deserialization, objectmapper, json]
image: assets/images/posts/2020-02-14/custom-deserialization.png
toc: true
---

This post try to resolve a problem where given a REST service with a already defined API, we want to add new API that uses the same service and without implement new logic.

## The problem

Let's suppose we have a REST service to register an object Person in our system and this receives a request with this JSON structure:

```
{
    "fullName": "...."
}

```

Now imagine we have an old client which is invoking other service to create Person but with different API request:

```
{
    "full_name": "...."
}

```

In the case we want the old client uses our service, we have to be compatible. For this case we have two different options:

* Create a new entry in our controller creating a new model compatible with the old client.
* Keep the same request object and modify the **deserialization** process in order to be compatible with the old client. This post will treat about this case.

> NOTE: For this example, we suppose that the old client is sending a header to difference with the current request model.

## Creating a custom deserializer

We need a custom deserializar for transform the old format in our current request model. Our custom deserializer will get the `full_name` field and will return the resquest with this value setted.

```java
public class PersonRequestCustomDeserializer {
    
    public PersonRequest deserialize(JsonParser jsonParser) throws IOException {
        
        JsonNode jsonNode = jsonParser.getCodec().readTree(jsonParser);
        String fullName = jsonNode.get("full_name").textValue();
        
        return PersonRequest.builder().fullName(fullName).build();
    }
}
```

## Defining a delegate

The delegate will be the responsible for select the correct deserializer depending on the header that we have received. 

If we receive the header "custom-api" with some value then the delegate will use the custom deserializer. Otherwise it will use default deserializer.

```java
public class PersonDelegatingDeserializer extends DelegatingDeserializer {
    
    private final PersonRequestCustomDeserializer personRequestCustomDeserializer = new PersonRequestCustomDeserializer();
    
    public PersonDelegatingDeserializer(JsonDeserializer defaultJsonDeserializer) {
        super(defaultJsonDeserializer);
    }
    
    @Override
    public Object deserialize(JsonParser jp, DeserializationContext dc) throws IOException {
        if (MDC.get(CUSTOM_API) == null) {
            return super.deserialize(jp, dc);
        } else {
            return personRequestCustomDeserializer.deserialize(jp);
        }
    }
    
    @Override
    protected JsonDeserializer<?> newDelegatingInstance(JsonDeserializer<?> jsonDeserializer) {
        return jsonDeserializer;
    }
}
```

> NOTE: We have used MDC to store the header. This header is setted using a `HandlerInterceptorAdapter` that is invoked before the deserializer. See `RequestsHandlerInterceptorAdapterConfig` in the code for more details.

## Registering custom deserializer in Spring

The last step is to register in Spring our custom delegate. To do this we have to add a SimpleModule in `MappingJackson2HttpMessageConverter` which is the class responsible for the conversions in controllers.

```java
@Configuration
@EnableWebMvc
public class MvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RequestsHandlerInterceptorAdapterConfig());
    }
    
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        for (HttpMessageConverter<?> converter : converters) {
            if (converter instanceof MappingJackson2HttpMessageConverter) {
                MappingJackson2HttpMessageConverter jacksonMessageConverter = 
                (MappingJackson2HttpMessageConverter) converter;
                
                ObjectMapper objectMapper = jacksonMessageConverter.getObjectMapper();
                SimpleModule simpleModule = new SimpleModule();
                simpleModule.setDeserializerModifier(new PersonRequestBeanDeserializerModifier());
                objectMapper.registerModule(simpleModule);

                break;
            }
        }
    }
}
```

The SimpleModule receives a deserializar modifier that is a class that connect with our deserializer delegate.

```java
public class PersonRequestBeanDeserializerModifier extends BeanDeserializerModifier {
    
    @Override
    public JsonDeserializer<?> modifyDeserializer(DeserializationConfig dc, BeanDescription bd,
            JsonDeserializer<?> deserializer) {
                
        if (PersonRequest.class.equals(bd.getBeanClass())) {
            return new PersonDelegatingDeserializer(deserializer);
        }
        return super.modifyDeserializer(dc, beanDesc, deserializer);
    }
}
```

## Testing the project

Standard request:

```
curl -X POST \
    -d '{"fullName":"test"}' \
    -H "Content-Type: application/json" \
    localhost:8080/person
```

Old request:

```
 curl -X POST \
    -d '{"full_name":"test"}' \
    -H "Content-Type: application/json" \
    -H "custom-api: true" \
    localhost:8080/person
```

## References

[1] [Link to the project in Github](https://github.com/frandorado/spring-projects/tree/master/spring-custom-serializer-deserializer)

