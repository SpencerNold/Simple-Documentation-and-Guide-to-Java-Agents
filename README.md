# How to Edit Running Java Programs
** Note: Do not use this information to violate terms of service of an application **
This repository is meant to serve as step-by-step documentation on Java agents. Agents can be used to load classes at runtime to a program, find byte sizes of Java objects, manipulate classes as they are being used, and in many other interesting and complex ways. This information should ONLY be used in projects which explicitly allow this, or in hobby projects/personal projects.
<h2>How does this work?</h2>
What we are going to do is create a Java agent, with specific permissions, attach it to a running jar file, and then from there we can transform the bytecode of classes, whether they have been loaded or not.
<h2>Creating an Agent</h2>
Creating an agent is fairly simple. We just need to create a Java project with an agent main class. For this, I am going to put the agent main class in a package named example, and name the class Agent.

```java
import java.lang.instrument.Instrumentation;

public class Agent {

    public static void agentmain(String args, Instrumentation instrumentation) {
        // TODO Add functionality to out agent
    }
}
```

Agents have the option of having a `premain` entrypoint, but for our purposes, this is unnecessary, and will just be using the `agentmain` entrypoint. the Instrumentation class allows us to manipulate the jar that the agent is attached to in the future. When we compile our agent into a jar, we must specify that this is an agent in the `META-INF/MANIFEST.MF`. For this, we can remove
<h2>Loading your Agent</h2>
To load your Agent, you must create a separate project, which I will call Loader.

```java
public class Loader {

	public static void main(String[] args) {
		
	}
}
```

When loading a Java agent, you can do this 2 different ways, with the `attach` method in the `VirtualMachine` class, or with a command line argument. We intend to attach our agent to a running Java program, rather than a program which we have programmed to attach an agent. As we are attempting to edit java bytecode dynamically, we will be using the handy `VirtualMachine.attach` method.
<h2>Attaching to a running process</h2>
To find a Java process running on any JVM on our machine, we can use the `VirtualMachine` class. In order to access this class, you may need to disable `com.sun.*` type filters on your IDE of choice.

```java
for (VirtualMachineDescriptor vm : VirtualMachine.list()) {
	
}
```

The `VirtualMachineDescripter` class has two methods which we can use, `displayName` and `id`. The `displayName` function returns a String representation of how the Java application was run. If the program was run with the `-jar` option, it will include the name of the jar file and the command-line arguments. If it was run with the `-classpath` option, it will return the name of the main class, the entrypoint of the jar. Attaching an agent with the sun attach libraries works as follows:

```java
import java.io.File;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

public class Loader {

	public static void main(String[] args) throws Exception {
		for (VirtualMachineDescriptor vmd : VirtualMachine.list()) {
			if (vmd.displayName().startsWith("class.name.Here")) { // Should be the command-line arguments for how the program was started. If with the -cp argument, it will have the main class name, and if it were with the -jar argument, it will contain the name of the Jar file.
				VirtualMachine vm = VirtualMachine.attach(vmd.id());
				vm.loadAgent(new File("path/to/your/agent.jar"), "this is the argument that will be passed into your agentmain method");
				vm.detach();
			}
		}
	}
}
```
!!! IMPORTANT !!! Using the `attach` method dynamically at runtime can throw quite a few errors, and it is important to properly handle these errors. For simplicity, we will just be adding a blanket `throws` clause to our `main` method.

Optionally, we can add a String argument to the `attach` method, which will get passed to the agent in the args parameter of our agentmain agent entrypoint.
<h2>Compiling our agent</h2>
Once we have our loader written, we can compile our agent jar file. Agents are compiled just as any other Java program is, only we must change the `META-INF/MANIFEST.MF` file inside our agent jar to include these lines as well as the normal mandatory values.

```
Agent-Class: example.Agent
Can-Retransform-Classes: true
```

<h2>Transforming classes</h2>
In order to retransform classes at runtime, we can use the `ClassFileTransformer` interface. We can either have a class implement this interface, or use an anonymous inner class.

```java
ClassFileTransformer transformer = new ClassFileTransformer() {
	public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
		return ClassFileTransformer.super.transform(loader, className, classBeingRedefined, protectionDomain, classfileBuffer);
	}
};
```

The `classfileBuffer` parameter is the bytecode bytes of the class being transformed, where the return value will be the transformed bytecode of the class. We can use Java ASM in order to easily interpret the bytecode, and write it back out as a `byte[]` (the returned value of the method). We finally then need to register the transformer with the `Instrumentation` class in the entrypoint method of our agent, and transform the class(es) we wish to change.

```java
instrumentation.addTransformer(transformer);
instrumentation.retransformClasses(classToTransform);
instrumentation.removeTransformer(transformer);
```

<h2>Launching</h2>
Finally, we run the application we wish to change. Launching the Loader project attaches our agent, which modifies the running java program.
