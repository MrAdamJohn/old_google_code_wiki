# Introduction

This not-quite-a-design-document sketches an approach to both data binding and dealing with server side business objects from a GWT client. It builds on the discussion at ValueStoreAndRequestFactory and RequestFactorySketch2, and like them is too cryptic but better than nothing. For the first time, I think I have a complete proposal here, at least behind my eyeballs.

Work is underway to build a prototype illustrating this scheme, which soon will find its way into the [bikeshed](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/README).

This was originally posted as a [wave](https://wave.google.com/wave/#restored:wave:googlewave.com!w%252BKEONTvE6C), and some discussion may be happening there.

rjrjr

# Details

I've backed away from the [souped up EventBus approach](RequestFactorySketch2.md) posted last week and am back to data being pushed directly from the ValueStore to HasValue and HasValueList subscribers. The new piece of the puzzle is Path objects, which accompany request objects to the server and tell it just which data we actually want returned.

Still unaddressed is how plain old client side Beans / model objects can take advantage of the data binding here without drowning them in boilerplate, but I take it as a given that we can and should make that work. It also seems likely that a lot of the builder style runtime construction that's happening here could (should) be happening at compile time instead. TBD.

So: imagine a bit of UI that wants to display data for an employee. You might make that happen with code a bit like this:

```
ExpensesRequestFactory requests = ...;
Employee e = ...; // from previous request
HasValue<String> myName = new TextField();

requests.values.subscribe(
  requests.employees().findEmployee(e.getId())
    .newPathThrough(Employee.DISPLAY_NAME)
    .to(myName));
```

Here's what's going on. `ExpensesRequestFactory` is an interface generated by a JPA-aware tool that trawled through the server side domain classes. The user then GWT.create()s an instance of it. A request factory provides builders that can create request objects corresponding to every service method on the server side domain layer.

(The intention is that we provide the tool. Presumably we could provide similar tools for other persistence frameworks. JPA just happens to be a good choice for GAE.)

In this example, `employees().findEmployee(e.getId())` is creating an instance of a request object. That's important: findEmployee() isn't an rpc call, it's a factory call to create an instance of something like FindEmployeeRequest. Later on that will go over the wire to be interpreted by the server, probably as part of a batch at the end of the current eventloop.

`Employee` is another product of our tool, and embodies the id and properties of a server side entity class, e.g. `Employee.DISPLAY_NAME`.  See a full fledged example of it in the blip below.

`requests.values.subscribe()` is a call to `ValueStore#subscribe(Path)`, which does the following:

  * It establishes a subscription keyed from the last entity + property of the path to an appropriate HasValue. In this case, myName is subscribed to Employee e's DISPLAY\_NAME.
  * It fires off the request that anchors the path (findEmployee()) to ensure the necessary property value is available, perhaps short circuiting if such a call is already cached.
  * Once we have the DISPLAY\_NAME value, it is pushed to myName.setValue().
  * Whenever e's DISPLAY\_NAME is updated it is pushed again, until the subscription is dropped.

If we can assume that every entity class has an appropriate findFoo() method, this can be simpler:

```
requests.values.subscribe(e
  .newPathThrough(Employee.DISPLAY_NAME)
  .to(myName));
```

It's not hard to imagine an annotation that would cause the above to be generated. (Pretend GWT.create() has actually been made useful and now accepts helper arguments.)

```
class EmployeeDisplay extends Composite implements HasValue<Employee> {
  private static final UiBinder UI_BINDER =
    GWT.create(UiBinder.class, EmployeeDisplay.class);
  private static final HasValueSupport DATA_BINDER =
    GWT.create(HasValueSupport.class, EmployeeDisplay.class);

  // works w/data binder b/c field name matches property
  // If it didn't match, could add @Path("displayName")
  @UiField HasValue<String>displayName; 

  private ValueStore values;

  public void setValueStore(ValueStore values) { this.values = values; }
  public void setValue(Employee value) {
    DATA_BINDER.setValue(this, values, value);
  }
}
```

Here's a more interesting path, to show someone's boss's name. Note that the subscription will be keyed directly to the boss entity, not to the e.MANAGER.DISPLAY\_NAME path.

```
HasValue<String> bossName = new TextField();

requests.values.subscribe(e
  .newPathThrough(Employee.MANAGER)
  .through(Employee.DISPLAY_NAME)
  .to(bossName));
```

For list values, we assume a HasValueList interface, which looks a lot like John L's ListHandler.  (I'm cheating a bit, would really need both display name and id in the ListBox for this to be useful, but just to get the idea across):

```
HasValueList<String> employeeList = new ListBox();

requests.values.subscribe(
  requests.employees().findAllEmployees()
    .forProperty(Employee.DISPLAY_NAME)
    .newPathTo(employeeList));
```

When you need more than just one field for an object (or object list), the `Values<E>` interface comes into play. It's basically a dedicated getter than can wrap any object managed by the value store:

```
HasValueList<Values<Employee>> employeeTable = new MyTableModel();

requests.values.subscribe(
  requests.employees().findAllEmployees()
    .forProperty(Employee.DISPLAY_NAME)
    .forProperty(Employee.USER_NAME)
    .newPathTo(employeeTable));
```

# Employee.java

```
@ServerType(com.google.gwt.sample.valuebox.domain.Employee.class)
public class Employee implements Entity {

  private final String id;
  private final Integer version;

  Employee(String id, Integer version) {
    this.id = id;
    this.version = version;
  }

  @javax.validation.constraints.Size(min = 2, max = 30)
  public static final Property<Employee, String> USER_NAME =
      new Property<Employee, String>(Employee.class, String.class);

  @javax.validation.constraints.Size(min = 2, max = 30)
  public static final Property<Employee, String> DISPLAY_NAME =
      new Property<Employee, String>(Employee.class, String.class);

  @javax.validation.constraints.NotNull
  public static final Property<Employee, Employee> SUPERVISOR =
      new Property<Employee, Employee>(Employee.class, Employee.class);

  public String getId() {
    return id;
  }

  public Integer getVersion() {
    return version;
  }
}
```

# Entity.java

```
public interface Entity {
  Object getId();
  Comparable<?> getVersion();
}
```

# HasValueList.java

```
public interface HasValueList<V> {
  void editValueList(boolean replace, int index, List<V> newValues);
  void setValueList(List<V> newValues);

  // @jlabanca: What does exact mean?
  void setValueListSize(int size, boolean exact); 
}
```

# Values.java

```
public interface Values<E> {
  E getEntity();
  
  <V, P extends Property<E, V>> V get(P property);
}
```