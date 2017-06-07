# PowerShell, Remoting y la Seguridad

Aunque PowerShell Remoting ha existido desde aproximadamente 2010, muchos administradores y organizaciones no pueden aprovecharse de ello, debido en gran parte a las políticas anticuadas o desinformadas de seguridad y prevención de riesgos. Este capítulo está diseñado para ayudar a abordar algunos de ellos, al proporcionar detalles técnicos honestos sobre cómo funcionan estas tecnologías. De hecho, presentan un riesgo significativamente menor que muchos de los protocolos de gestión y comunicaciones que ya están en uso generalizado; los protocolos más antiguos se benefician principalmente de estar "anclados" en políticas, pero nunca examinados de cerca.

## Ni PowerShell ni Remoting son una "puerta trasera" para el Malware

Este es un gran error. Tenga en cuenta que de forma predeterminada, PowerShell no ejecuta secuencias de comandos. Cuando lo hace, sólo puede ejecutar comandos que el usuario ejecutor tiene permiso para ejecutar - no ejecuta nada bajo una cuenta super-privilegiada, y tampoco omite ni los permisos existentes ni la seguridad. De hecho, como PowerShell está basado en .NET, es improbable que algún autor de malware se moleste en utilizar PowerShell. Tal atacante podría simplemente llamar a la funcionalidad de .NET Framework directamente mucho más fácilmente.

De forma predeterminada, PowerShell Remoting sólo permite que los administradores se conecten y, una vez conectados, sólo pueden ejecutar comandos con permisos para ejecutarlos, sin posibilidad de omitir permisos o seguridad subyacente. A diferencia de las herramientas anteriores que funcionaban bajo una cuenta altamente privilegiada (como LocalSystem), PowerShell Remoting ejecuta los comandos impersonando al usuario que envió los comandos.

Conclusión: Debido a la forma en que funciona, PowerShell Remoting no permite que ningún usuario, autorizado o no, haga algo que no pueda hacer a través de una docena de otros medios, incluido el inicio de sesión en la consola. Cualquier protección que usted tenga en su lugar para prevenir ese tipo de ataques (como mecanismos apropiados de autorización y autenticación) también protegerá a PowerShell y a Remoting. Si permite a los administradores iniciar sesión en las consolas de servidor, ya sea físicamente o mediante el Escritorio remoto, tiene una exposición de seguridad mucho mayor que la que realiza a través de PowerShell Remoting.

Además, PowerShell ofrece una mejor oportunidad para limitar incluso a los administradores. Un EndPoint Remoting (o la configuración de la sesión) se puede modificar para permitir que sólo los usuarios especificados se conecten a él. Una vez conectado, el EndPoint puede restringir más los comandos que esos usuarios pueden ejecutar. Esto proporciona una oportunidad mucho mejor para la administración delegada. En lugar de hacer que los administradores inicien sesión en las consolas y hagan lo que les plazca, puede hacer que se conecten a EndPoints restringidos y seguros y que sólo completen las tareas específicas que el EndPoint permite

## PowerShell Remoting is Not Optional

As of Windows Server 2012, PowerShell Remoting is enabled by default and is mandatory for server management. Even when running a graphical management console locally on a server, the console still "goes out" and "back in" via Remoting to accomplish its tasks. Without Remoting, server administration is impossible. Organizations are therefore well-advised to start immediately finding a way to include Remoting in their permitted protocols. Otherwise, critical services will not be able to be managed, even through Remote Desktop or directly on the server console.

This approach actually helps better secure the data center. Because local administration is exactly the same as remote administration (via Remoting), there's no longer any reason to physically or remotely access server consoles. The consoles can thus remain more locked down and secured, and Administrators can stay out of the data center entirely.

## Remoting Does Not Transmit or Store Credentials

By default, Remoting uses Kerberos, an authentication protocol that does not transmit passwords across the network. Instead, Kerberos relies on passwords as an encryption key, ensuring that passwords remain safe. Remoting can be configured to use less-secure authentication protocols (such as Basic), but can also be configured to require certificate-based encryption for the connection.

Further, Remoting never stores credentials in any persistent storage by default. A Remote machine never has access to a user's credentials; it has access only to a delegated security token (a Kerberos "ticket"). That is stored in volatile memory which cannot, by OS design, be written to disk - even to the OS page file. The server presents that token to the OS when executing commands, causing the command to be executed with the original invoking user's authority - and nothing more.

## Remoting Uses Encryption

Most Remoting-enabled applications apply their own encryption to their application-level traffic sent over Remoting. However, Remoting can also be configured to use HTTPS (certificate-encrypted connections), and can be configured to make HTTPS mandatory. This encrypts the entire channel using high-level encryption, while also ensuring mutual authentication of both client and server.

## Remoting is Security-Transparent

As stated, Remoting neither adds anything to, nor takes anything away from, your existing security configuration. Remote commands are executed using the delegated credentials of whatever user invoked the commands, meaning they can only do what they have permission to do - and what they could presumably do through a half-dozen other tools anyway. Whatever auditing you have in place in your environment cannot be bypassed by Remoting. Unlike many past "remote execution" solutions, Remoting does not operate under a single "super-privileged" account unless you expressly configure it that way (which requires several steps and cannot possibly by accomplished accidentally, as it requires the creation of custom endpoints).

Remember: Anything someone can do via Remoting, they can already do in a half-dozen other ways. Remoting simply provides a more consistent, controllable, and scalable means of doing so.

## Remoting is Lower Overhead

Unlike Remote Desktop Connection (RDC, which many Administrators currently use to manage remote servers), Remoting is very low-overhead. It does not require the server to spin up an entire graphical operating environment, impacting server performance and memory management. Remoting is also more scalable, enabling authorized users (mainly Administrators in most cases) to execute commands against multiple servers at once - which improves consistency and reduces error, while also speeding up response times and lowering administrative overhead.

Remoting is Microsoft's way forward. To not use Remoting is to deliberately attempt to use Windows in a way that it was explicitly designed not to do. You will reduce, not improve your security, while also increasing operational overhead, enabling greater instance of human error, and reducing server performance. Microsoft Administrators have for decades been toiling under an operational paradigm that was wrong-headed and short-sighted; Remoting is finally delivering to Windows the administrative model that every other network operating system has used for years, if not decades.

## Remoting Uses Mutual Authentication

Unlike nearly every other remote management technique out there - including tools like PSExec and even, under some circumstances, Remote Desktop, PowerShell Remoting by default requires mutual authentication. The user attempting to connect to a server is authenticated and known; the system also ensures that the server connected to is the intended server and not an imposter. This provides far better security than past techniques, while also helping to reduce error - you can't "accidentally log on to the wrong console" as you could if you just walked into the data center.

## Summary

At this point, denying PowerShell Remoting is like denying Ethernet: It's ridiculous to think you'll successfully operate your environment without it. For the first time, Microsoft has provided a supported, official, baked-in technology for remote server administration that does not use elevated credentials, does not store credentials in any way, that supports mutual authentication, and that is complete security-transparent. This is the administration technology we should have had all along; moving to it will only make your environment more manageable and more secure, not less.

