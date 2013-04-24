##Testing the Android way
After learning the benefits of a test-driven style as a Rails developer, you never want to go back to the old style of writing code that could possibly work, possibly not! As a newcomer to Android, finding the right strategies and tools to drive tests was the first task at hand. After digging in, I found several excellent libraries that help facilitate a test-driven workflow on android, and a test culture that currently lags behind rails, but that is rapidly adopting the use of automated testing.

###Robolectric
The largest barrier when getting started on this challenge was the dependency of the android core libraries themselves upon the actual android operating system. The [AndroidTestCase](http://developer.android.com/reference/android/test/AndroidTestCase.html) classes provided by Google, for example, need a new instance of an emulator to run.
This emulator then needs a new instance of the APK under test on it.. The whole cycle of spinning up the emulator, deploying the APK and running the actual test could take a few minutes upon every run of the suite to get set up - a major time sink.

This slow rate of speed to run tests was a dealbreaker, and i felt it would likely reduce the chances we'd actively use the tests! Fortunately for us, the [Robolectric Project](http://pivotal.github.io/robolectric/) from Pivotal Labs has done the hard work of removing this dependency. 

Effectively, **Robolectric** replaces the behavior of code that would otherwise require an emulator or actual device with it's own, and once set up, we can write jUnit-style tests against our classes without needing the baggage of the emulator! It does this using so-called "**Shadow Classes**", mock implementations of android core libraries. 

**Robolectric** also goes further than this, enabling you to provide your own custom behavior for these shadow classes! This is extremely helpful for custom test behavior you may need. For example, my current project depends upon loading large sets of data to the sqlite database managed by android. Using the "**Shadow Class**" notion, **Robolectric** allowed me to implement a 'test fixture' style database reset, where any changes that may have happened to a default set of data i provide gets reset to ensure I can assert results dependent upon this data. Here's what i mean:

<pre>
public class MyTestRunner extends RobolectricTestRunner{
  @Override
  public void beforeTest(Method method) {
    super.beforeTest(method);
    // swaps in custom implementations of the sqlite android database class.
    Robolectric.bindShadowClass(MyShadowSQLiteDatabase.class);
  }
}
</pre>

and then in the test itself:
<pre>
@RunWith(MyTestRunner.class)
public class TestDatabaseStuff extends BaseAttTest{

  public static final String TEST_DATABASE = "test/fixtures/test_database.s3db";
  public static final String ORIG_DATABASE = "test/fixtures/test_database.orig";

  @Before
  public void setup() throws IOException{
    //reset the test fixture
    originalDatabase = new File(ORIG_DATABASE);
    fileToWrite = new File(TEST_DATABASE);
    FileUtils.copyFile(originalDatabase, fileToWrite);
  }
}
</pre>

The **MyShadowSQLiteDatabase** implementation provides custom behavior that points to our custom test database. 
Without **Robolectric**, we would be dependent upon the core android class behavior.

###Syntactic Sugar
Having a way to run tests fast, i then began to look for ways to write tests faster and more cleanly. 

####Fest for Android
The first discovery was Fest-Android [http://square.github.io/fest-android/], a library extension for FEST framework [http://fest.easytesting.org/] is a great improvement beyond the regular jUnit-style assertions from the fine folks at Square Labs. It gives a chainable (or "fluent") syntax for checking  assertions, and makes tests easier to write (and read). A bonus of this is that any decent java IDE can code-complete the available assertions for any property - cleaning up the confusion the "expected" and "actual" syntax in the jUnit style test syntax.

for example:
<pre>
//regular junit:
assertEquals(View.GONE, view.getVisibility());
//fest!
assertThat(view).isGone();
</pre>

At the point **assertThat(view)** is called, the IDE can then give intelligent feedback about the types of assertions available.

###Mockito = rspec-mocks..sort of
For more complicated classes, overriding complicated elaborate functionality that is coupled to other classes to isolate tests is done using the notion of 'mocks'.
The [Mockito](https://code.google.com/p/mockito/) framework for java does this well. An example from a project was to selectively change the behavior of a method to return a timestamp i could test against, rather than a dynamically generated one. Mockito made this a snap:

`Mockito.doReturn((long) 1363027600).when(myQueryObject).getCurrentTime();`

Whenever **myQueryObject.getCurrentTime** is called, a predefined value is returned instead of the current! Very useful for having classes return results you can actually test!

###Robotium = Integration Tests
Sometimes, unit tests alone fail to address behaviour you will want to check with your tests. 
For example, let's say I want to assert that entering a valid username & password, clicking the login button, and then clicking on the account details button takes me to the right view within my application. A unit test doesn't really capture this "path" through the application, and is better described using an "integration test", or a test that checks multiple components of the system work properly in conjunction with one-another. For this type of work, I found [Robotium](https://code.google.com/p/robotium/), which uses a selenium-like style to run the test from the UI. This of course requires an emulator or device to work. I found that keeping the unit test project and integration test projects seperate was a good compromise between the fast robolectric tests, and the sometimes necessary robotium tests to test overall correct behavior across the system.

###Dependency Injection
To reduce the amount of effort required to set up classes for tests, the use of Dependency Injection is a huge win. Simply put, it allows you to configure the instantiation of classes across your application without manually doing so. It also allows the configuration of special behavior for classes without writing code from scratch. [Roboguice](https://github.com/roboguice/roboguice) brings the google GUICE injection framework to android, and makes it possible to use dependency injection throughout your android app. Check out the Roboguice wiki on how to use this great tool: [Roboguice Wiki](https://github.com/roboguice/roboguice/wiki)


###Wrapping up
While test-driven isn't as pervasive with Android, it seems doing things the TDD way is gaining momentum within the Android community. I hope this article helped to give a good overview of some of the tools that are out there for testing your android project with! Don't hesitate to ask any questions about the article, or point out any other tools i should know about below!