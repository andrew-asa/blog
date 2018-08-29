+ JsonDeserializer接口
```
public abstract T deserialize(JsonParser p, DeserializationContext ctxt)
         throws IOException, JsonProcessingException;
//p.getCurrentValue(); 可以获取当前上下文环境
//ctxt.setAttribute(k,v); 可以
```
