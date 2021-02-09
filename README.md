
#  Fn + GraalVM Together

> As you make your way through this tutorial, look out for this icon. ![user
input](images/userinput.png) Whenever you see it, it's time for you to perform
an action.

This tutorial will focus on a specific GraalVM capability:  Native Image
ahead-of-time (AOT) compilation for Java functions.

First, create an app as follows:

![user input](images/userinput.png)
>```sh
> fn create app myapp
>```

```
Successfully created app:  myapp
```

Second, create Java a function as follows:

![user input](images/userinput.png)
>```sh
> fn init --init-image fnproject/fn-java-native-init graalvmfn
>```

```
Creating function at: ./graalvmfn
Building from init-image: fnproject/fn-java-native-init
func.yaml created.
```

If you compare this to the approach used for generating a “regular” Java
function, the key difference is that we instruct the Fn CLI to use the
`fnproject/fn-java-native-init` Docker init-image (see
[here](https://medium.com/fnproject/even-wider-language-support-in-fn-with-init-images-a7a1b3135a6e)
for more details on init-images) to generate a boilerplate GraalVM-based Java
function, instead of relying on the standard Java runtime.

While the standard Java FDK uses the default Java runtime, our *GraalVM
native-image* function relies on the Docker runtime (which also explains the
presence of a Dockerfile).

You will notice that one of the steps is using *GraalVM’s native-image* utility
to compile the Java function ahead-of-time into a native executable.

Let's build the function and deploy it to the app we created earlier. We can do
this with a single command as follows. To see in details what is going on, you
can add the —-verbose flag.

![user input](images/userinput.png)
>```sh
> cd graalvmfn
> fn --verbose deploy --local --app myapp
>```

```
Deploying function at: ./graalvmfn
Deploying graalvmfn to app: myapp
Bumped to version 0.0.2
Building image fndemouser/graalvmfn:0.0.2
FN_REGISTRY:  fndemouser
Current Context:  default

Sending build context to Docker daemon  18.43kB
Step 1/22 : FROM fnproject/fn-java-fdk-build:latest as build
 ---> 43818d2b84e5
...
```

The resulting function is a self-contained executable that does not require a
Java virtual machine to run.  It includes your function code, dependent
libraries, and JDK classes compiled into native machine code linked with all the
necessary platform libraries and a thin runtime layer called GraalVM Substrate
that provides application services including garbage collection and memory
management.  The resulting executable is quite small as it only contains the
code you need to run your function.  All unused classes, methods, and even
fields are removed at compile time.  An added feature of this approach is that
the attack surface area of an executable is much smaller than a corresponding
Java app running on a JVM.

![native-build](images/native-build.png)

Once generated, the native function executable is added to a lightweight base
image (busybox:glibc) with some related dependencies. This will constitute the
function Docker image that the Fn infrastructure will use when the function is
invoked.

```
...
Step 14/19 : FROM busybox:glibc
 ---> 31266219dbbb
Step 15/19 : WORKDIR /function
 ---> Using cache
 ---> febbf69a130a
Step 16/19 : COPY --from=build-native-image /function/func func
 ---> 1a189b8feaed
Step 17/19 : COPY --from=build-native-image /function/runtime/lib/* .
 ---> 62014a94da38
Step 18/19 : ENTRYPOINT ["./func", "-XX:MaximumHeapSizePercent=80"]
 ---> Running in 3cf5123e47b2
Removing intermediate container 3cf5123e47b2
 ---> b9fa9a85ac56
Step 19/19 : CMD [ "com.example.fn.Graalvmfn::handleRequest" ]
 ---> Running in 2fc889a46131
Removing intermediate container 2fc889a46131
 ---> af722ba17bca
Successfully built af722ba17bca
Successfully tagged fndemouser/graalvmfn:0.0.2
```

You can check the size of the resulting function image using Docker.

>```
> docker images | grep graalvmfn
>```

```
fndemouser/graalvmfn            0.0.2               d5742bc560f2        22 minutes ago      20.6MB
```

As you can see, the function image that includes everything required to run,
including the operating system and the function native executable, is only
around 21 MB!

Finally, let's invoke the deployed function and time how long it takes:

![user input](images/userinput.png)
>```sh
> time fn invoke myapp graalvmfn
>```

```
Hello, world!

real	0m0.770s
user	0m0.034s
sys	0m0.007s
```

For comparison, let's again create a function using the standard Java runtime:

![user input](images/userinput.png)
>```
> cd ..
>fn init --runtime java jvmfn
>```

```
Creating function at: ./jvmfn
Function boilerplate generated.
func.yaml created.
```

Next, we deploy the app:

![user input](images/userinput.png)
>```
>cd jvmfn
>fn --verbose deploy --local --app myapp
>```


Again, you can check the size of the resulting function image using Docker:

>```
> docker images | grep jvmfn
>```

And you'll see that it's much larger:

```
fndemouser/jvmfn            0.0.2               a86a4d09bece        2 minutes ago      222MB
```

And now let's invoke it and see how long it takes:

![user input](images/userinput.png)
>```
> time fn invoke myapp jvmfn
>```

```
Hello, world!

real	0m1.998s
user	0m0.031s
sys	0m0.016s
```

We can see that the execution time of a GraalVM Native Image generated
executable function is much faster in comparison. These numbers will vary
depending on the machine you run the tests on but this basic benchmark shows
that the cold startup of the GraalVM native-image function is significantly
faster. In this particular example, the cold startup time of the GraalVM
native-image function (770ms) is over twice as fast as the cold startup time
(1998ms) of the same Java function that uses a regular JVM (HotSpot in this
case).

### Conclusions

As you can see, using GraalVM with Fn for Serverless Java function is simple and
straightforward. The Fn GraalVM integration relies on GraalVM’s Native Image
feature to ahead-of-time compile a Java function into a native executable that
embeds the function itself plus the necessary components of a runtime like the
garbage collector, resulting in a lightweight function Docker image with faster
startup-time. Using GraalVM in Fn gives us Java functions with performanc on par
with functions written in languages that are compiled natively such as Go! In
Fn, GraalVM offers all the benefits of Java functions, including support of the
Java FDK, with the performance of native code!

The same function you built in this tutorial could also be seamlessly deployed
to the [Oracle
Functions](https://docs.cloud.oracle.com/iaas/Content/Functions/Concepts/functionsoverview.htm)
cloud platform. The smaller size of the docker image built using GraalVM would
provider further performance benefits on Oracle Functions, since the container
must be pulled by the service when scaling up the function.