# Gestión de sesiones

Cuando crea una conexión Remoting entre dos máquinas, está creando una sesión en la terminología de PowerShell. Hay un número increíble de opciones que se pueden aplicar a estas sesiones, y en esta parte de la guía los guiaremos a través de ellas.

## Sesiones Ad-Hoc vs. Persistentes

Cuando utiliza un comando Remoting, principalmente Invoke-Command o Enter-PSSession, y especifica un nombre de equipo utilizando su parámetro -ComputerName, está creando una sesión Ad Hoc. Básicamente, PowerShell crea una sesión, la utiliza, y luego ejecuta sus comandos, todo de forma automática.

De manera alternativa, puede utilizar New-PSSession para crear explícitamente una nueva sesión, que luego puede utilizarse pasándola como el parámetro -Session de Invoke-Command, Enter-PSSession y muchos otros comandos compatibles con Remoting. Cuando crea manualmente una sesión, es su responsabilidad deshacerse de ella cuando haya terminado de utilizarla. Sin embargo, si tiene una sesión abierta y cierra su instancia de PowerShell, esa sesión se eliminará automáticamente por usted, por lo que no estaría dejando nada pendiente que necesita ser limpiado.

## Desconexión y Reconexión de Sesiones

En PowerShell v3, puede desconectar y volver a conectar sesiones utilizando Disconnect-PSSession y Connect-PSSession. Estos comandos aceptan cada uno un objeto de sesión, que normalmente crearía con New-PSSession.

Una sesión desconectada deja una copia de PowerShell en funcionamiento en el equipo remoto. Esta es una buena manera de ejecutar una tarea de larga duración, desconectarse y luego volver a conectarse más tarde para comprobar su estado. Incluso puede desconectar una sesión en una computadora, moverse a otra computadora y volver a conectarse a esa sesión \(aunque no puede conectarse a la sesión desconectada de otro usuario porque está limitado a volver a conectarse a la suya\).

Por ejemplo, la figura 5.1 muestra una sesión que se está creando desde un cliente a un servidor. A la sesión se le asigna una tarea para realizarla como un trabajo de fondo y, a continuación, se desconecta la sesión. Es importante tener en cuenta que el comando y el trabajo de fondo están en el servidor \(DC01\), no en el cliente.

![image070.png](images/image070.png)

Figure 5.1: Creating, using, and disconnecting a session

In figure 5.2, we've moved to a different machine. We're logged on, and running PowerShell, as the same user that we were on the previous client computer. We retrieve the session from the remote computer, and then reconnect it. We then enter the newly reconnected session, display that background job, and receive some results from it. Finally, we exit the remote session and shut it down via Remove-PSSession.

![image071.png](images/image071.png)

Figure 5.2: Reconnecting to, utilizing, and removing a session

Obviously, disconnected sessions can present something of a management concern, because you're leaving a copy of PowerShell up and running on a remote machine - and you're doing so in a way that makes it difficult for someone else to even see you've done it! That's where session options come into play.

## Session Options

Whenever you run a Remoting command that creates a session - whether persistent or ad-hoc - you have the option of specifying a -SessionOption parameter, which accepts a PSSessionOption object. The default option object is used if you don't specify one, and that object can be found in the built-in $PSSessionOption variable. It's shown in figure 5.3.

![image072.png](images/image072.png)

Figure 5.3: The default PSSessionOption object stored in $PSSessionOption

As you can see, this specifies a number of defaults, including the operation timeout, idle timeout, and other options. You can change these by simply creating a new session option object and assigning it to $PSSessionOption; note that you need to do this in a profile script if you want your changes to become the new default every time you open a new copy of PowerShell. Figure 5.4 shows an example.

![image073.png](images/image073.png)

Figure 5.4: Creating a new default PSSessionOption object

Of course, a 2-second idle timeout probably isn't very practical (and in fact won't work - you must specify at least a 60-second timeout in order to use the session object at all), but you'll note that you only need to specify the option parameters that you want to change - everything else will go to the built-in defaults. You can also specify a unique session option for any given session you create. Figure 5.5 shows one way to do so.

![image074.png](images/image074.png)

Figure 5.5: Creating a new PSSessionOption object to use with a 1-to-1 connection

By specifying intelligent values for these various options, you can help ensure that disconnected sessions don't hang around and run forever and ever. A reasonable idle timeout, for example, ensures that the session will eventually close itself, even if an administrator disconnects from it and subsequently forgets about it. Note that, when a session closes itself, any data within that session - including background job results - will be lost. It's probably a good idea to get in the practice of having data saved into a file (by using Export-CliXML, for example), so that an idle session doesn't close itself and lose all of your work.

