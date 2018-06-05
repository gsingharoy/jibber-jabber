
# How to add defaults for missing properties in Play Scala JSON Serialization

I came across this interesting use case, when I was trying to unmarshall some JSON into case classes and I wanted to take into account missing properties or keys. Since I have been using golang prior to using scala, and this was quite given in golang and it automatically took care of missing properties. Let’s dive into the following example to find out what exactly I am talking about.

Consider the case class

```scala
case class FooClass(field1: String = "unknown", field2: Int = -1)

object FooClass {
  implicit def jsonFormat = Json.format[FooClass]
}
```

I have a case class called FooClass and it has a couple of fields. I have defined some default values for both the fields. So now say you want to unmarshall the JSON

    {"field1": "bar", field2: 42}

It would Unmarshall this to FooClass("bar", 42) . All good till now. Imagine, you have stored this JSON blob directly in a SQL database. Now, if I introduce a new field, say field3 , then all the existing entries will not be able to be unmarshalled because they would have the missing property or key, field3 . So my case class now looks like

```scala
case class FooClass(
  field1: String = "unknown",
  field2: Int = -1,
  field3: Boolean = false)
```

What I would now want is for my previous JSON to be automatically unmarshalled to FooClass("bar", 42, false) .

Now let’s find out how I can achieve this

### Play 2.6 to the rescue

Play 2.6 already ships with WithDefaultValues and this can easily be used to implicitly add the default values for the missing properties.

So simply add this in your companion object

```scala
object FooClass{
  implicit def jsonFormat = Json.using[Json.WithDefaultValues].format[FooClass]
}
```

Easy right ? Unfortunately my project was not on Play 2.6 and I could not upgrade to 2.6 yet due to a lot of library conflicts.

### Reads.pure()

Many people would argue that we should introduce the new fields as Option and read them asNullable or asOpt from the json reads. But in our example it is a boolean field, then having it Option[Boolean] sounds a little smelly.

So the next solution is to use pure . So our companion object becomes

```scala
object FooClass {
  import play.api.libs.json._
  import play.api.libs.functional.syntax._

  implicit def jsonReads = (
    ((__ \ "field1").read[String] or Reads.pure("unknown")) and
    ((__ \ "field2").read[Int] or Reads.pure(-1)) and
    ((__ \ "field3").read[Boolean] or Reads.pure(false))
  )((field1, field2, field3) => FooClass(field1, field2, field3))

  implicit def jsonWrites = Json.writes[FooClass]
}
```

This is relatively clean but you still need to manually find each field and fallback with the Reads.pure . Of course you can clean up the above code so that you do not need to define the defaults in two places. But what if you have a huge case class and its quite deeply nested. You will end up with writing a lot of boiler plate code. So let’s check out the solution which I came up with.

### Merge with default JSON

Here is the solution which I finally ended up using. Please note that, it can work for only such use cases where you can have a default class object, and yes, this is a hack ! For cases where you have a class say BarClass below, where it is not possible to define a default object for the BarClass , then this solution will not work.

```scala
case class BarClass(
  field1: String,
  field2: Int,
  field3: Boolean = **false**)
```

Now if we consider a JSON as a map then

    {"field1": "unknown", "field2": -1, "field3": false}
    +
    {"field1": "bar", "field2": 42}
    =
    {"field1": "bar", "field2": 42, "field3": false}

All it is doing is simply merging the properties or keys of both the maps and the first map contains all the properties. Any property which is present in the second map is overwritten. Now the resulting map or the JSON object contains all the missing fields with the default values as well and can be unmarshalled without any errors.

Let’s jump into the code.

```scala
import play.api.libs.json._

/**
 * Extend this trait by the companion objects of your models where
 * you would like to seamlessly read missing paths in the JSON.
 * Note : This can only work if you have a default value for your case class.
 *
 *
 * @tparam T The class which is extending this trait
 */
trait JsonDefaultValueFormatter[T] {

  /**
   *
   * @return An empty default object with default values.
   */
  def defaultObject: T

  /**
   *
   * @return A default JSON reads method. If you have a class with primitive types,
   *         then simply use Json.reads[T]
   */
  def defaultReads: Reads[T]

  /**
   *
   * @return A JSON writes method.
   */
  implicit def jsonWrites: Writes[T]

  /**
   * This is the implicit which will be used by the Object to Unmarshall.
   * It works with a simple logic that first it builds a json object of a default object and merges with
   * the json object to be unmarshalled. In the process any missing keys are automatically added.
   * Then the resulting json object is unmarshalled using the defaultReads method.
   *
   * @return A json.Reads[T] method
   */
  implicit def jsonReads: Reads[T] = new **Reads[T] {
    def reads(json: JsValue): JsResult[T] = {
      val defaultJson = Json.toJson(defaultObject)
      val finalJson = Seq(defaultJson, json).foldLeft(Json.obj())((obj, a) => obj.deepMerge(a.as[JsObject]))
      defaultReads.reads(finalJson)
    }
  }
}
```

I have defined a trait called JsonDefaultValueFormatter which contains the helper methods. The implicit method jsonReads is doing this merge for me. It is first creating a JSON object of the default object of the class and then doing a deep merge with the actual JSON object. Let us look a bit closely into the abstract methods which it requires. First is the defaultObject . Here you need to define the default object value. We will simply use this to marshall it to a JSON object in thejsonReads method. Then is the defaultReads method. This we need to define how to read your class. Similarly we have a jsonWrites as well.

Now let us use this for our FooClass . I would need to extend this trait in the Companion object

```scala
object FooClass extends JsonDefaultValueFormatter[FooClass] {
  import play.api.libs.json._

  val defaultObject: FooClass = FooClass()
  val defaultReads: Reads[FooClass] = Json.reads[FooClass]
  implicit def jsonWrites: Writes[FooClass] = Json.writes[FooClass]
}
```

I simply have defined my defaultObject to be FooClass() since all of my fields in the case class have a default. Similarly am using default writes and reads for JSON. But you can have your own methods there if you have such a requirement.

This solution worked pretty well for me since I had a very huge deeply nested case class and I always had a default object for the case classes affected. I did not need to write a lot of boiler plate code, and I can introduce new fields later on easily and yet have my existing JSON objects stored in my DB to be backward compatible. Also because of the deepMerge usage, it works fine with deeply nested case classes as well.

Do let me know if you find these solutions useful. If you have any other solution to tackle this, then please do write in the comments.
