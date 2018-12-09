+ JsonDeserializer接口
```
public abstract T deserialize(JsonParser p, DeserializationContext ctxt)
         throws IOException, JsonProcessingException;
//p.getCurrentValue(); 可以获取当前上下文环境
//ctxt.setAttribute(k,v); 可以
```

# 注解
+ @JsonInclude(JsonInclude.Include.NON_DEFAULT) 默认属性不返回，比如类里面有a字段，那么 如果a字段为null 则返回给前端的时候json里面没有a字段
