```java
public class Teacher {
    private String name;
    private String age;
    private List<Student> studentList;

}
```

```java
public class Student {
    private String name;
    private String age;
}
```

```Java
public static void main(String[] args){
    Student student = new Student();
        student.setAge("12");
        student.setName("adv");
        Teacher teacher = new Teacher();
        teacher.setAge("34");
        teacher.setName("sds");
        List<Student> studentList = new ArrayList<>(1);
        studentList.add(student);
        teacher.setStudentList(studentList);
}
```

```java
public class ReflectList {
    public static void getListItem(Object obj) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException {
        Field[] fields = obj.getClass().getDeclaredFields();
        for (Field field : fields) {
            if (!field.isAccessible()){
                field.setAccessible(true);
            }
            if (List.class.isAssignableFrom(field.getType())){
                Type t = field.getGenericType();
                ParameterizedType pt = (ParameterizedType) t;
                Class clz = (Class) pt.getActualTypeArguments()[0];//Student Class
                Class clazz = field.get(obj).getClass(); // List Class
                Method m = clazz.getDeclaredMethod("size");
                int size = (int) m.invoke(field.get(obj));
                for (int i = 0; i < size; i++) {
                    Method getMethod = clazz.getDeclaredMethod("get", int.class);
                    if (!getMethod.isAccessible()){
                        getMethod.setAccessible(true);
                    }
                    System.out.println(getMethod.invoke(field.get(obj), i));
                }

                if (Student.class.isAssignableFrom(clz)){
                    List<Student> list = (List<Student>) field.get(obj);//List<Student>
                    for (Student student : list) {
                        System.out.println(student.getAge() + student.getName());
                    }
                }
            }


        }

    }
}
```

