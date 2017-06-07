# Fundamentos de Remoting

Windows PowerShell 2.0 introdujo una potente tecnología, Remoting, refinada y ampliada en PowerShell 3.0. Basada principalmente en protocolos y técnicas estandarizadas, el sistema de Remoting es posiblemente uno de los aspectos más importantes de PowerShell: los futuros productos de Microsoft se basarán en él casi en su totalidad para las comunicaciones administrativas a través de una red.

Desafortunadamente, Remoting es también un sistema complejo de componentes, y mientras que Microsoft ha intentado proporcionar la dirección sólida para usarla en una variedad de escenarios, muchos administradores todavía luchan con esta. Este "mini e-book" está diseñado para ayudarle a entender mejor lo que es el Remoting, cómo funciona y, lo que es más importante, cómo usarlo en una variedad de situaciones diferentes.

**Nota** Tenga en cuenta que esta guía no pretende reemplazar la gran variedad de libros existentes que cubren los fundamentos de Remoting, como el propio _Learn Windows PowerShell in a Month of Lunches_ (http://morelunches.com) de Don Jones o _PowerShell in Depth_. En su lugar, esta guía complementa a aquellas que proporcionan instrucciones paso a paso para muchos de los escenarios "alrededor" de un sistema de comunicación remota, e intenta explicar algunos de los comportamientos y requerimientos de los sistemas remotos más inusuales.

## ¿Qué es Remoting?

En esencia, el acceso remoto le permite acceder a máquinas remotas a través de una red y recuperar datos o ejecutar código en una o varias computadoras remotas. Esto no es una idea nueva. Ya en el pasado una serie de diferentes tecnologías remotas han intentado lo mismo. Algunos Cmdlets de PowerShell han proporcionado tradicionalmente capacidades propias de acceso remoto limitadas, mientras que la mayoría de los Cmdlets no admiten la conexión remota por su propia cuenta.

Con PowerShell Remoting se encuentra finalmente un entorno genérico que permite la ejecución remota para, literalmente, cualquier comando que se puede ejecutar en una instanica de PowerShell de forma local. Por lo que en lugar de agregar capacidades de acceso remoto a cada Cmdlet y/o aplicación, simplemente se deja a PowerShell transferir la ejecución de su código al equipo de destino y a continuación, enviar los resultados de vuelta.

A lo largo de este libro nos centraremos en el control remoto de PowerShell pero no cubriremos las funciones remotas privadas no estándar incorporadas en algunos Cmdlets seleccionados
.

## Examinando la arquitectura de Remoting

Como se muestra en la figura 1.1, la arquitectura remota genérica de PowerShell se compone de numerosos componentes y elementos diferentes e interrelacionados.

![image003.png](images/image003.png)

Figura 1.1: Los elementos y componentes de PowerShell Remoting

Aquí está la lista completa:

- En la parte inferior de la figura está su computadora, o más correctamente su cliente. Es donde usted se sienta físicamente, y donde iniciará la mayor parte de sus actividades de control remoto. 

- Su computadora se comunicará a través de WS-MAN, o del protocolo de servicios web para la administración. Este es un protocolo basado en http(s) que puede encapsular una variedad de tipos de comunicación. Hemos ilustrado el uso de  http, que es la configuración predeterminada, pero también podría ser fácilmente https

- En el equipo remoto, en la terminología adecuada, el servidor (que no hace referencia al sistema operativo), ejecuta el servicio de administración remota de Windows (WinRM). Este servicio está configurado para tener uno o más oyentes. Cada oyente espera el tráfico entrante de WS-MAN en un puerto específico, cada uno ligado a un protocolo específico (http o https), y en direcciones IP específicas (o todas las direcciones locales)

- Cuando un oyente recibe tráfico, el servicio WinRM busca el EndPoint a donde se debe enviar el  tráfico. Para nuestro propósito, un EndPoint usualmente estará asociado con una instancia de Windows PowerShell. En términos de PowerShell, un EndPoint también se denomina una configuración de sesión. Esto se debe a que además de lanzar PowerShell, se pueden cargar secuencias de comandos y módulos, agregar restricciones sobre lo que puede hacer el usuario conectado y aplicar configuraciones adicionales de sesión específicas que no se mencionan aquí.

**Nota** Aunque mostramos **powershell.exe** en nuestro diagrama, eso solo para propósitos de ilustración. **Powershell.exe** es la aplicación de consola de PowerShell, y no tendría sentido tener esta ejecución como un proceso de fondo en un equipo remoto. El proceso real se denomina **wsmprovhost.exe**, que aloja PowerShell en segundo plano para conexiones remotas.

Como se puede ver, un único equipo remoto puede tener fácilmente decenas o incluso cientos de EndPoints, cada uno con una configuración diferente. PowerShell 3.0 configura tres EndPoints por defecto: uno para PowerShell de 32 bits (en sistemas de 64 bits), un EndPoint de PowerShell por defecto (que es de 64 bits en sistemas x64) y otro para PowerShell Workflow. Comenzando con Windows Server 2008 R2, hay un cuarto EndPoint predeterminado para las tareas de Server Manager Workflow.

## Habilitando Remoting

La mayoría de las versiones cliente de Windows, iniciando con Windows Vista, no habilitan las conexiones remotas entrantes de forma predeterminada, aunque las versiones de servidor más recientes de Windows vienen con Remoting habilitado. El primer paso con Remoting suele ser habilitarlo en los equipos en que se desean recibir conexiones entrantes. Hay tres maneras de habilitar Remoting. La tabla 1.1 compara lo que se puede lograr con cada una de ellas.

Tabla 1.1 comparando las maneras de habilitar Remoting

|  | Enable-PSRemoting | Política de grupo | Manualmente paso a paso |
| --- | --- | --- | --- |
| Establecer WinRM para auto-iniciar e iniciar el servicio | Si | Si | Si - utilice **Set-Service** y **Start-Service**. |
| Configurar el detector de HTTP | Si | Puede configurar el registro automático de Listeners, sin crear Listeners personalizados | Si - Utilice la utilidad de línea de comandos WSMAN y la unidad **WSMAN:** de PowerShell |
| Configurar el detector de HTTPS | No | No | Si - Utilice la utilidad de línea de comandos WSMAN y la unidad **WSMAN:** de PowerShell |
| Configurar EndPoints / Configurar sesiones | Si | No | Si - utilice el Cmdlet PSSessionConfiguration |
| Configurar la excepción de Firewall de Windows | Si\* | Si\* | Si\* - Utilice Cmdlets del Firewall o la GUI del Firewall de Windows |

**Nota** Tenga en cuenta que las versiones existentes de cliente de Windows, como Windows Vista, no permiten excepciones de Firewall en una red identificada como "pública". Las redes deben ser "casa" o "trabajo/dominio" para permitir excepciones. En PowerShell 3.0, se puede ejecutar **Enable-PSRemoting** con el modificador **-SkipNetworkProfileCheck** para evitar este problema..

Estaremos habilitando la administración remota en nuestro entorno de prueba ejecutando **Enable-PSRemoting**. Es rápido, fácil e incluye todo lo necesario. También vera una gran cantidad de tareas manuales a realizar en las siguientes secciones.

## Entorno de pruebas

Usaremos un entorno de pruebas consistente en las siguientes secciones. Consiste en seis máquinas virtuales en _cloudshare.com_ configuradas como se muestra en la figura 1.2.

![image004.png](images/image004.png)

Figura 1.2: configuración del entorno de pruebas

Algunas notas importantes:

- .NET Framework v4 y PowerShell 3.0 están instalados en todos los equipos. La mayor parte de lo que cubriremos también se aplica a PowerShell 2.0.

- Como se muestra, la mayoría de las computadoras tienen un nombre de computadora numérico (c2108222963, y así sucesivamente); El controlador de dominio para cada dominio (que también es un servidor DNS) tiene registros CNAME con nombres más fáciles de recordar.

- Cada controlador de dominio tiene un reenviador condicional configurado para el otro dominio, de modo que las máquinas de cualquiera de los dominios puedan resolver nombres de equipos en el otro dominio.

- Realizamos todas las tareas como miembro del grupo de administradores del dominio, a menos que se indique lo contrario.

- Creamos un sexto servidor completamente independiente que no está en ningún dominio. Esto será útil para cubrir algunas de las situaciones que no son de dominio con las que puede encontrarse en un sistema de comunicación remota.

**Tenga cuidado** al abrir PowerShell en un equipo que tenga habilitado el control de cuenta de usuario (UAC), asegúrese de hacer clic con el botón derecho en el icono de PowerShell y seleccione “Ejecutar como administrador”. Si la barra de título de la ventana PowerShell resultante no comienza con la palabra Administrador: entonces no tiene privilegios administrativos. Puede comprobar los permisos de forma programática con esto _(whoami /all | select-string S-1-16-12288) -ne $null_ en una consola de PowerShell. En un Shell con permisos de administrador se devuelve **True**, de lo contrario será **False**.

## Habilitando Remoting

Comenzamos ejecutando Enable-PSRemoting en las seis computadoras. Debemos asegurarnos que el comando finaliza sin errores. Cualquier error en este punto es una señal para se detenga y resuelva el error antes de intentar continuar. La figura 1.3 muestra la salida esperada.

![image005.png](images/image005.png)

Figura 1.3: salida esperada de Enable-PSRemoting

**Nota**: Observara un uso desmedido de capturas de pantalla a lo largo de esta guía. Me permiten asegurar que no cometo errores ortográficos o  errores del tipo copiar/pegar. Verá exactamente lo que escribimos y los resultados de su ejecución.

Ejecutar Get-PSSessionConfiguration debe revelar los tres o cuatro EndPoints creados por Enable-PSRemoting. La figura 1.4 muestra la salida esperada en uno de los servidores.

![image006.png](images/image006.png)

Figura 1.4: Salida esperada de Get-PSSessionConfiguration

**Nota**: la figura 1.4 ilustra que se puede esperar que diferentes EndPoints se configuren en diferentes máquinas. Este ejemplo fue con un servidor Windows 2008 R2 equipo, que tiene menos EndPoints que una máquina con Windows 2012.

Vale la pena tomar un momento para comprobar rápidamente la configuración de Remoting. Para los equipos que forman parte del mismo dominio, al iniciar sesión como administrador de dominio de ese dominio, el sistema de comunicación remota debería "funcionar". Compruébelo rápidamente conectarse de una computadora a otra usando Enter-PSSession. 

**Nota**: en otros entornos, es posible que una cuenta de administrador de dominio no sea la única que pueda usar Remoting. Si en su hogar o entorno de trabajo se tienen cuentas adicionales en el grupo de administradores locales como estándar en su dominio, también podrá utilizar esas cuentas para realizar llamadas remotas.

La figura 1.5 muestra la salida esperada, en la que también ejecutamos un comando dir rápido y luego salimos de la sesión remota.

![image007.png](images/image007.png)

Figura 1.5: comprobación de la conectividad remota desde el cliente al controlador de dominio DCA.

**Precaución**: si está configurando su propio entorno de pruebas, no continúe hasta que haya confirmado la conectividad de los sistemas remotos entre dos equipos del mismo dominio. Por ahora no necesitamos comprobar otros escenarios.

## Tareas “core” de Remoting

PowerShell proporciona dos escenarios principales para el uso de Remoting. El primero, Remoting 1-a-1, es similar en su naturaleza  al shell SSH disponible en sistemas UNIX y Linux. Con él, obtendrá acceso a una línea de comandos en un único equipo remoto. El segundo, Remoting  1-a-muchos, permite enviar un comando (o una lista de comandos) en paralelo, a un conjunto de equipos remotos. Hay otro par de técnicas secundarias útiles que veremos más adelante.

### Remoting 1-a-1 
El comando Enter-PSSession se conecta a un equipo remoto y permite el acceso a una línea de comandos en ese equipo. Puede ejecutar cualquier comando en dicho equipo, siempre y cuando tenga permiso para realizar esa tarea. Tenga en cuenta que no está creando un inicio de sesión interactivo. La conexión se auditará como un inicio de sesión de red, al igual que si se estuviera conectando al recurso compartido administrativo C$ de la computadora. PowerShell no cargará ni procesará las secuencias de comandos del perfil de usuario en el equipo remoto. Cualquier script que elija ejecutar (y esto incluye la importación de módulos de script) sólo funcionará si la política de ejecución de la máquina remota lo permite.

```
Enter-PSSession -computerName DC01
```

**Nota**: Mientras esté conectado a una máquina remota a través de Enter-PSSession, el “prompt” de la línea de comandos cambia y muestra el nombre del sistema remoto al que está conectado entre corchetes. Si ha personalizado su “prompt”, todas esas personalizaciones se perderán porque el “prompt” se creará en el sistema remoto y se transferirá de regreso a usted. Todas las entradas de teclado se envían a la máquina remota, y todos los resultados son devueltos a usted. Es importante tener en cuenta esto porque no puede utilizar Enter-PSSession en un script. Si lo hace, el script seguiría ejecutándose en su máquina local, ya que no se ingresó ningún código (en el teclado) de forma interactiva.

### 1-to-Many Remoting

With this technique, you specify one or more computer names and a command (or a semicolon-separated list of commands); PowerShell sends the commands, via Remoting, to the specified computers. Those computers execute the commands, serialize the results into XML, and transmit the results back to you. Your computer deserializes the XML back into objects, and places them in the pipeline of your PowerShell session. This is accomplished via the Invoke-Command cmdlet.

```
Invoke-Command -computername DC01,CLIENT1 -scriptBlock { Get-Service }
```

If you have a script of commands to run, you can have Invoke-Command read it, transmit the contents to the remote computers, and have them execute those commands.

```
Invoke-Command -computername DC01,CLIENT1 -filePath c:\Scripts\Task.ps1
```

Note that Invoke-Command will, by default, communicate with only 32 computers at once. If you specify more, the extras will queue up, and Invoke-Command will begin processing them as it finishes the first 32. The -ThrottleLimit parameter can raise this limit; the only cost is to your computer, which must have sufficient resources to maintain a unique PowerShell session for each computer you're contacting simultaneously. If you expect to receive large amounts of data from the remote computers, available network bandwidth can be another limiting factor.

### Sessions

When you run Enter-PSSession or Invoke-Command and use their -ComputerName parameter, Remoting creates a connection (or session), does whatever you've asked it to, and then closes the connection (in the case of an interactive session created with Enter-PSSession, PowerShell knows you're done when you run Exit-PSSession). There's some overhead involved in that set-up and tear-down, and so PowerShell also offers the option of creating a persistent connection - called a PSSession. You run New-PSSession to create a new, persistent session. Then, rather than using -ComputerName with Enter-PSSession or Invoke-Command, you use their -Session parameter and pass an existing, open PSSession object. That lets the commands re-use the persistent connection you'd previously created.

When you use the -ComputerName parameter and work with ad-hoc sessions, each time you send a command to a remote machine, there is a significant delay caused by the overhead it takes to create a new session. Since each call to Enter-PSSession or Invoke-Command sets up a new session, you also cannot preserve state. In the example below, the variable $test is lost in the second call:

```
PS> Invoke-Command -computername CLIENT1 -scriptBlock { $test = 1 }
PS> Invoke-Command -computername CLIENT1 -scriptBlock { $test }
PS>
```

When you use persistent sessions, on the other hand, re-connections are much faster, and since you are keeping and reusing sessions, they will preserve state. So here, the second call to Invoke-Command will still be able to access the variable $test that was set up in the first call

```
PS> $Session = New-PSSession -ComputerName CLIENT1
PS> Invoke-Command -Session $Session -scriptBlock { $test = 1 }
PS> Invoke-Command -Session $Session -scriptBlock { $test }
1
PS> Remove-PSSession -Session $Session
```

Various other commands exist to check the session's status and retrieve sessions (Get-PSSession), close them (Remove-PSSession), disconnect and reconnect them (Disconnect-PSSession and Reconnect-PSSession, which are new in PowerShell v3), and so on. In PowerShell v3, you can also pass an open session to Get-Module and Import-Module, enabling you to see the modules listed on a remote computer (via the opened PSSession), or to import a module from a remote computer into your computer for implicit Remoting. Review the help on those commands to learn more.

Note: Once you use New-PSSession and create your own persistent sessions, it is your responsibility to do housekeeping and close and dispose the session when you are done with them. Until you do that, persistent sessions remain active, consume resources and may prevent others from connecting. By default, only 10 simultaneous connections to a remote machine are permitted. If you keep too many active sessions, you will easily run into resource limits. This line demonstrates what happens if you try and set up too many simultaneous sessions:

```
PS> 1..10 | Foreach-Object { New-PSSession -ComputerName CLIENT1 }
```

## Remoting Returns Deserialized Data

The results you receive from a remote computer have been serialized into XML, and then deserialized on your computer. In essence, the objects placed into your shell's pipeline are static, detached snapshots of what was on the remote computer at the time your command completed. These deserialized objects lack the methods of the originals objects, and instead only offer static properties.

If you need to access methods or change properties, or in other words if you must work with the live objects, simply make sure you do so on the remote side, before the objects get serialized and travel back to the caller. This example uses object methods on the remote side to determine process owners which works just fine:

```
PS> Invoke-Command -ComputerName CLIENT1 -scriptBlock { Get-WmiObject -Class Win32_Process | Select-Object Name, { $_.GetOwner().User } }
```
Once the results travel back to you, you can no longer invoke object methods because now you work with "rehydrated" objects that are detached from the live objects and do not contain any methods anymore:

```
PS> Invoke-Command -ComputerName CLIENT1 -scriptBlock { Get-WmiObject -Class Win32_Process } | Select-Object Name, { $_.GetOwner().User }
```
Serializing and deserializing is relatively expensive. You can optimize speed and resources by making sure that your remote code emits only the data you really need. You could for example use Select-Object and carefully pick the properties you want back rather than serializing and deserializing everything.

## Enter-PSSession vs. Invoke-Command

A lot of newcomers will get a bit confused about remoting, in part because of how PowerShell executes scripts. Consider the following, and assume that SERVER2 contains a script named C:\RemoteTest.ps1:

```
Enter-PSSession -ComputerName SERVER2  
C:\RemoteTest.ps1
```

If you were to sit and type these commands interactively in the console window on your client computer, this would work (assuming remoting was set up, you had permissions, and all that). However, if you pasted these into a script and ran that script, it wouldn't work. The script would try to run C:\RemoteTest.ps1 _on your local computer. _

The practical upshot of this is that Enter-PSSession is really meant for _interactive use by a human being, _ not for batch use by a script. If you wanted to send a command to a remote computer, from within a script, Invoke-Command is the right way to do it. You can either set up a session in advance (useful if you plan to send more than one command), or you can use a computer name if you only want to send a single command. For example:

```
$session = New-PSSession -ComputerName SERVER2  
Invoke-Command -session $session -ScriptBlock { C:\RemoteTest.ps1 }
```

Obviously, you'll need to use some caution. If those were the _only_ two lines in the script, then when the script finished running, $session would cease to exist. That might disconnect you (in a sense) from the session running on SERVER2. What you do, and even whether you need to worry about it, depends a lot on what you're doing and how you're doing it. In this example, everything would _probably_ be okay, because Invoke-Command would "keep" the local script running until the remote script finished and returned its output (if any).


