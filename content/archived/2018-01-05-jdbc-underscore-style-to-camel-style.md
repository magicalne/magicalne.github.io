---
layout: post
title: JDBC， field binding， 驼峰 VS 下划线
date: 2018-01-05
tag:
- jdbc
- phoenix
---

项目引入phoenix之后，也一并引入了很多问题，那天估计脑子真的热了，居然花时间做了这样的一个功能；（

事情是这样的，phoenix是hbase上层的sql“解释器”，用来将sql翻译成对应的hbase scan，带来了很多副作用，但是，有一个好处，这是SQL！会让你少写很多模版代码，而且处理数据的方式更加标准。

JDBC这块，没有引入ORM框架，整个项目也没有使用到spring，项目角度更佳倾向轻量级。但是为了开发速度，引入了apache下的[commons-dbutils](https://commons.apache.org/proper/commons-dbutils/)。

这里有个小小的问题，像ORM框架一样，你在select的时候，如何将JDBC返回的数据映射到java中的entity呢？当然是按字段名称一一映射了啊。可是，数据库的字段都是下划线风格的，而java都是驼峰风格的。。。那天真是强迫症犯了啊。。。居然一定要做一个风格转换。。。

通用的解决方案就是用annotation，比如json parser，可以通过定义类似@JsonProperty("loan_id")，重新进行mapping。但是写annotation还是有点麻烦的，这里用了偷懒的方案。

dbutils支持通过自定义BeanProcessor来实现类似功能。而我这里的功能只需要将，大写的下划线风格转化成驼峰风格即可。

这里只需要继承BeanProcessor， 并实现方法：
>public int[] mapColumnsToProperties(ResultSetMetaData rsmd, PropertyDescriptor[] props) throws SQLException

```java
@Override
    public int[] mapColumnsToProperties(ResultSetMetaData rsmd,
                                        PropertyDescriptor[] props) throws SQLException {
        int cols = rsmd.getColumnCount();
        int[] columnToProperty = new int[cols + 1];
        Arrays.fill(columnToProperty, PROPERTY_NOT_FOUND);

        for (int col = 1; col <= cols; col ++) {
            String columnName = rsmd.getColumnLabel(col);
            if (null == columnName || 0 == columnName.length()) {
                columnName = rsmd.getColumnName(col);
            }
            String propertyName = columnToPropertyOverrides.get(columnName);
            if (propertyName == null) {
                propertyName = toCamelCase(columnName);
                columnToPropertyOverrides.put(columnName, propertyName);
            }
            for (int i = 0; i < props.length; i++) {
                if (propertyName.equalsIgnoreCase(props[i].getName())) {
                    columnToProperty[col] = i;
                    break;
                }
            }
        }

        return columnToProperty;
    }
```

这里的columnToPropertyOverrides是做的一层缓存，所以映射只需要执行一次就好。

不过，讲真，这种功能确实没啥用，如果当时冷静冷静，估计就不会有这段代码了...