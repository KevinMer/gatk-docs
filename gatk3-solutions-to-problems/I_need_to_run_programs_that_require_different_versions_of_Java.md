## I need to run programs that require different versions of Java

By KateN

<p>We sometimes need to be able to use multiple versions of Java on the same computer to run command-line tools that have different version requirements. At the time of writing, GATK requires an older version of Java (1.7), whereas Picard requires the most recent version (1.8). So in order to run both Picard tools and GATK tools on your computer, we present a solution for doing so that is reasonably painless.</p>

<p>You will need to have both versions of Java installed on your machine. The Java installation package for 1.8 can be found <a rel="nofollow" href="http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html">here</a>, and the package for 1.7 is <a rel="nofollow" href="http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html">here</a>. Note that we point to the “JDK” (Java Development Kit) packages because they are the most complete Java packages (suitable for developing in Java as well as running Java executables), and we have had reports that the “JRE” (Java Runtime Environment) equivalents were not sufficient to run GATK on some machines.</p>

<p>First, check your current default java version by opening your terminal and typing <code class="code codeInline" spellcheck="false">java -version</code>. If the version starts with “1.8”, you will need to add the following code to the beginning of your GATK command to specify that it should be run using version 1.7.</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/b7/ae060ff79d13c9cb403306b1a09a3e.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>If your default version starts with “1.7”, then you will need to prepend the code below to your Picard command:</p>

<p><img src="https://us.v-cdn.net/5019796/uploads/FileUpload/52/0c1f5b1284704c89e67257eb7d46fd.png" alt="image" class="embedImage-img importedEmbed-img"></img></p>

<p>You may need to change the orange part in each code snippet, which should refer to the specific version of java you have installed on your machine (version and update). To find that, simply navigate to the folder where you had installed the JDK. Under the “JavaVirtualMachines” folder, you should find JDK folders that name the specific version and update.</p>
