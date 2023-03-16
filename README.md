# How to Edit Running Java Programs
** Note: Do not use this information to violate terms of service of an application **
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
When loading a Java agent, you can do this 2 different ways, with the `attach` method in the `VirtualMachine` class, or with a command line argument. We intend to attach our agent to a running Java program, rather than a program which we have programmed to attach an agent. As a result, we can use byte-buddy-agent https://github.com/raphw/byte-buddy (v. 1.14.2 is the current version at the time of writing this) to attach an agent to a running jar.
<h2>Attaching to a running process</h2>
To find a Java process running on any JVM on our machine, we can use the `VirtualMachine` class. In order to access this class, you may need to disable `com.sun.*` type filters on your IDE of choice.
```java
for (VirtualMachineDescriptor vm : VirtualMachine.list()) {
	
}
```
The `VirtualMachineDescripter` class has two methods which we can use, `displayName` and `id`. The `displayName` function returns a String representation of how the Java application was run. If the program was run with the `-jar` option, it will include the name of the jar file and the command-line arguments. If it was run with the `-classpath` option, it will return the name of the main class, the entrypoint of the jar. We can then use the `attach` static method in the `ByteBuddyAgent` class, to attach our Java agent to the process. Our loader main class should look something like this now.
```java
import java.io.File;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import net.bytebuddy.agent.ByteBuddyAgent;

public class Loader {

	public static void main(String[] args) {
		for (VirtualMachineDescriptor vm : VirtualMachine.list()) {
			if (vm.displayName().startsWith("name of jar or main class name")) {
				ByteBuddyAgent.attach(new File("path/to/the/agent.jar"), vm.id());
			}
		}
	}
}
```
Optionally, we can add a String argument to the `attach` method, which will get passed to the agent in the args parameter of our agentmain agent entrypoint.
<h2>Compiling our agent</h2>
Once we have our loader written, we can compile our agent jar file. Agents are compiled just as any other Java program is, only we must change the `META-INF/MANIFEST.MF` file to include these lines as well as the mandatory values.
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
The `classfileBuffer` parameter is the bytecode bytes of the class being transformed, where the return value will be the transformed bytecode of the class. We can use Java ASM and ASM-Tree in order to easily interpret the bytecode, and write it back out as a `byte[]`, returning the class changed with ASM. We finally then need to register the transformer with the `Instrumentation` class in our entrypoint method.
```java
instrumentation.addTransformer(transformer);
instrumentation.retransformClasses(classToTransform);
instrumentation.removeTransformer(transformer);
```
<h2>Launching</h2>
Finally, we run the application we wish to change. Launching the Loader project attaches our agent, which modifies the running java program.