# General

* All code must be accompanied with sufficiently thorough test cases which validate the functionality.
* Test cases should be self-sufficient and not rely on data produced by other tests, or the outcome of other tests in any other way. A test case may, however, generate all it's resources via the `setUp` method; however, such generation should be done just once for all the tests.
* If tests are failing, they are a top priority.
* Pull requests will not be merged, if there are failing tests.
* Don't use REST API for testing your service class methods. Test them directly.
* Put `@Rollback(false)` if you want to persist something during the test execution. It will tell Spring to not to call `rollback()` on this class method transactions.
* Use [rest-assured](https://github.com/rest-assured/rest-assured/wiki/GettingStarted#spring-mock-mvc) for testing REST API. Don't try to re-involve the bicycle and inherit all rest-assured initialization stuff from `RestAssuredBaseTest`. Mark your test as `@IntegrationTest` and take a look at the existing examples (subclasses of `RestAssuredBaseTest`). If you would like to have some init method with `@Before` annotation make sure that you will have `super.init()` as a first line in such kind of method.  
* All tests **MUST be idempotent**, it means that we should be able to execute them multiple times from console or IDE and execution should not depends on how many times we ran this test.

# Artifact-related Tests

For artifact-related test you could create instance of TestCaseWithArtifactGeneration in your unit test or reuse one from `RestAssuredBaseTest`. Just delegate your call to super class field.

## Generating Artifacts

The [ArtifactGenerator](https://github.com/strongbox/strongbox/blob/master/strongbox-testing/strongbox-testing-core/src/main/java/org/carlspring/strongbox/artifact/generator/ArtifactGenerator.java) class provides various methods for generating valid Maven test artifacts.

To generate an artifact, you can use the following code:

    generateArtifact(REPOSITORY_BASEDIR_RELEASES.getAbsolutePath(),
                     "org.carlspring.strongbox.resolve.only:foo",
                     new String[]{ "1.1" }); // Used by testResolveViaProxy()

If you're generating this from your test's `setUp` method, please, make sure that:
* The artifact generation part is only invoked once
* For each artifact you add a comment explaining which test method is using this resource

## Deploying Artifacts

The [ArtifactDeployer](https://github.com/strongbox/strongbox/blob/master/strongbox-testing/strongbox-testing-core/src/main/java/org/carlspring/strongbox/artifact/generator/ArtifactDeployer.java) class provides a way to deploy Maven artifacts to a repository.

The following is an example of artifact deployment taken from the [ArtifactDeployerTest](https://github.com/strongbox/strongbox/blob/master/strongbox-testing/strongbox-testing-core/src/test/java/org/carlspring/strongbox/artifact/generator/ArtifactDeployerTest.java):

    private static ArtifactClient client;
    
    
    @Before
    public void setUp()
            throws Exception
    {
        if (!BASEDIR.exists())
        {
            //noinspection ResultOfMethodCallIgnored
            BASEDIR.mkdirs();

            client = new ArtifactClient();
            client.setUsername("maven");
            client.setPassword("password");
            client.setPort(assignedPorts.getPort("port.jetty.listen"));
            client.setContextBaseUrl("http://localhost:" + client.getPort());
        }
    }

    @Test
    public void testArtifactDeployment()
            throws ArtifactOperationException,
                   IOException,
                   NoSuchAlgorithmException,
                   XmlPullParserException
    {
        Artifact artifact = ArtifactUtils.getArtifactFromGAVTC("org.carlspring.strongbox:test:1.2.3");

        String[] classifiers = new String[] { "javadocs", "jdk14", "tests"};

        ArtifactDeployer artifactDeployer = new ArtifactDeployer(BASEDIR);
        artifactDeployer.setClient(client);
        artifactDeployer.generateAndDeployArtifact(artifact, classifiers, "storage0", "releases", "jar");
    }

_Notice_: if you are going to test REST API you need to create instance of `ArtifactDeployer` using superclass `buildArtifactDeployer()` method. It will inject appropriate instance of `RestClient` to it.

## Adding Artifacts To The Index

For test cases where you need to generate artifacts and add them to the index, please have a look at the [TestCaseWithArtifactGenerationWithIndexing](https://github.com/strongbox/strongbox/blob/master/strongbox-storage/strongbox-storage-indexing/src/test/java/org/carlspring/strongbox/testing/TestCaseWithArtifactGenerationWithIndexing.java) class.

Please, note that the above class is not currently something you can extend outside the scope of the `strongbox-storage-indexing` module.

# Testing REST calls

## How to write your own integration test

Here is sequence of actions for anyone who would like to write it's own REST API test.
* extend `RestAssuredBaseTest` class
* put `@IntegrationTest` and `@RunWith(SpringJUnit4ClassRunner.class)` on top of your class
* review existing examples (subclasses of `RestAssuredBaseTest`)

## How to use rest-assured

Here is the simplest example that will send HTTP GET request on /greeting endpoint:

    given().
    when().
    get("/greeting").
    then().
    statusCode(200);

`Notice`: you should have `import static com.jayway.restassured.module.mockmvc.RestAssuredMockMvc.*;` in your test.

## Where it differs from stock version of rest-assured

Instead of given() please use givenLocal() of `RestAssuredBaseTest`.

## Do I need to extend `RestAssuredArtifactClient` or write my own methods in unit tests?

Basically no. The only reason to extend that class is when you would like to reuse something between several unit tests (to avoid code duplication).

## Referencies

Please review this excellent article. It contains a lot of cool examples: [unit-testing-spring-mvc-controllers-with-rest-assured](https://blog.jayway.com/2014/01/14/unit-testing-spring-mvc-controllers-with-rest-assured/)

If you still have any questions please review original [usage wiki page by rest-assured](https://github.com/rest-assured/rest-assured/wiki/usage).