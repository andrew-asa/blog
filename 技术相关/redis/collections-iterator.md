# map.(keys,values,entryset).iterator
+ 直接上测试用例
```
@Test
    public void testKeysIterator() {

        Map<String, String> map = new HashMap<String, String>();
        map.put("1", "1");
        map.put("2", "2");
        map.put("3", "3");
        map.put("4", "4");
        Set<String> keys = map.keySet();
        Iterator<String> keysiter = keys.iterator();
        while (keysiter.hasNext()) {
            String next = keysiter.next();
            if ("2".equals(next)) {
                keysiter.remove();
            }
        }
        Assert.assertFalse(map.containsKey("2"));
    }

    @Test
    public void testValuesIterator() {

        Map<String, String> map = new HashMap<String, String>();
        map = new HashMap<String, String>();
        map.put("1", "1");
        map.put("2", "2");
        map.put("3", "3");
        map.put("4", "4");
        Collection<String> values = map.values();
        Iterator<String> valuesIters = values.iterator();
        while (valuesIters.hasNext()) {
            String next = valuesIters.next();
            if ("2".equals(next)) {
                valuesIters.remove();
            }
        }
        Assert.assertFalse(map.containsKey("2"));
    }


    @Test
    public void testEntrySetIterator() {

        Map<String, String> map = new HashMap<String, String>();
        map = new HashMap<String, String>();
        map.put("1", "1");
        map.put("2", "2");
        map.put("3", "3");
        map.put("4", "4");
        Set<Map.Entry<String, String>> entrySet = map.entrySet();
        Iterator<Map.Entry<String, String>> entryIterator = entrySet.iterator();
        while (entryIterator.hasNext()) {
            Map.Entry<String, String> next = entryIterator.next();
            if ("2".equals(next.getKey())) {
                entryIterator.remove();
            }
        }
        Assert.assertFalse(map.containsKey("2"));
    }
```
+ 三个测试用例都是通过的
+ 说明在map这种基础的数据结构的时候当获取keys,values,entryset的时候不能只是简单的把所有的key，values，key，value item 获取出来简单扔到相应的集合就可以，因为这样没有办法确保这些集合的迭代器删除的时候把map对应的元素进行删除。
+ 所有的集群集合实现都有这样的一个问题，这也是集群集合实现的一个难点，redis可以在scan等类似方法上面进行进一步的开发，但是到ehcache里面这就又是一个难点了。
