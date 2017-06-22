# Trabajar con Endpoints (también conocido como Configuraciones de Sesión)

Como aprendió al principio de esta guía, Remoting está diseñado para trabajar con múltiples puntos finales distintos en un equipo. En la terminología de PowerShell, cada punto final es una configuración de sesión o simplemente una configuración. Cada uno puede ser configurado para ofrecer servicios y capacidades específicos, así como tener restricciones y limitaciones específicas.

## Conexión a un punto final diferente

Cuando utiliza un comando como Invoke-Command o Enter-PSSession, normalmente se conecta al punto final predeterminado de un equipo remoto. Eso es lo que hemos hecho hasta ahora. Pero puede ver los otros puntos finales habilitados ejecutando Get-PSSessionConfiguration, como se muestra en la figura 3.1.

![image042.png](images/image042.png)

Figura 3.1: Listando los puntos finales instalados

**Nota:** Como señalamos en un capítulo anterior, cada computadora mostrará puntos finales diferentes por defecto. Nuestra salida era de un equipo con Windows Server 2008 R2, que tiene menos puntos finales predeterminados que, por ejemplo, un equipo con Windows 2012.

Cada punto final tiene un nombre, como "Microsoft.PowerShell" o "Microsoft.PowerShell32". Para conectarse a un punto final específico, agregue el parámetro -ConfigurationName al comando Remoting, como se muestra en la Figura 3.2.

![image043.png](images/image043.png)

Figura 3.2: Conexión a una configuración específica (punto final) por nombre

## Creación de un punto de extremo personalizado

Existen varias razones para crear un punto final personalizado (o una configuración):

- Puede tener scripts y módulos de carga automática cada vez que alguien se conecta.
- Puede especificar un descriptor de seguridad (SDDL) que determina quién tiene permiso para conectarse.
- Puede especificar una cuenta alternativa que se utilizará para ejecutar todos los comandos dentro del punto final, en lugar de utilizar las credenciales de los usuarios conectados.
- Puede limitar los comandos que están disponibles para los usuarios conectados, restringiendo así sus capacidades.

Hay dos pasos para configurar un punto final: Crear un archivo de configuración de sesión que definirá las capacidades de los puntos finales y luego registrar ese archivo, que habilita el punto final y define sus configuraciones. La Figura 3.3 muestra la ayuda para el comando New-PSSessionConfigurationFile, que realiza el primero de estos dos pasos.

![image044.png](images/image044.png)

Figura 3.3: El comando New-PSSessionConfigurationFile

Aquí algunos de los parámetros que le permiten especificar (revise el archivo de ayuda por los otros parámetros):

- -Path: El único parámetro obligatorio, es la ruta y el nombre de archivo del archivo de configuración que creará. Ingrese un nombre y utilice una extensión .PSSC para el nombre de archivo.

- -AliasDefinitions: Esta es una tabla hash de alias y sus definiciones. Por ejemplo, @ {Name = 'd'; Definition = 'Get-ChildItem'; Options = 'ReadOnly'} definiría el alias d. Utilice una lista separada por comas de estas tablas hash para definir varios alias.

- -EnvironmentVariables: Una tabla hash única de variables de entorno para cargar en el punto final: @{'MyVar'='\SERVER\Share';'MyOtherVar'='SomethingElse'}

- -ExecutionPolicy: Por defecto es Restricted si no especifica otra cosa. Utilice Unrestricted, AllSigned o RemoteSigned. Establece la directiva de ejecución de secuencias de comandos para el punto final.

- -FormatsToProcess y -TypesToProcess: Cada una de estas es una lista separada por comas de la ruta de acceso y los nombres de los archivos a cargar. El primero especifica los archivos .format.ps1xml que contienen definiciones de vista, mientras que el segundo especifica un archivo .ps1xml para el ETS (Extensible Type System) de PowerShell.

- -FunctionDefinitions: Una lista separada por comas de tablas hash, cada una de las cuales define una función para aparecer dentro del punto final. Por ejemplo,  @{Name='MoreDir';Options='ReadOnly';Value={ Dir | more }}

- -LanguageMode: El modo para el lenguaje de script de PowerShell. "FullLanguage" y "NoLanguage" son las opciones. Este último sólo permite ejecutar funciones y Cmdlets. También hay "RestrictedLanguage" que permite un subconjunto muy pequeño del lenguaje de scripting para trabajar - vea la ayuda para más detalles.

- -ModulesToImport: Una lista de nombres de módulos separados por comas para cargar en el punto final. También puede utilizar tablas hash para especificar versiones de módulo específicas. Lea la ayuda completa del comando para obtener más detalles.

- -PowerShellVersion: '2.0' o '3.0', especifica la versión de PowerShell que desea que el punto final utilice. 2.0 sólo se puede especificar si PowerShell v2 se instala independientemente en el equipo que aloja el punto final (instalando v3 "en la parte superior de" v2 permite que v2 continúe existiendo).

- -ScriptsToProcess: Una lista separada por comas de nombres de rutas y archivos de secuencias de comandos que se ejecutan cuando un usuario se conecta al punto final. Puede usar esto para personalizar el espacio de ejecución del punto final, definir funciones, cargar módulos o hacer cualquier otra cosa que un script pueda hacer. Sin embargo, para ejecutar la directiva de ejecución de secuencias de comandos debe permitir la secuencia de comandos.

- -SessionType: " Empty " no carga nada por defecto, dejándolo a usted libre de cargar lo que quiera a través de scripts o los parámetros de este comando. "Default" carga las extensiones principales de PowerShell normales, además de cualquier otra cosa que haya especificado a través del parámetro. "RestrictedRemoteServer" agrega una lista fija de siete comandos, además de lo que haya especificado. Consulte la ayuda para obtener detalles sobre lo que se ha cargado

**Caution:** Some commands are important - like Exit-PSSession, which enables someone to cleanly exit an interactive Remoting session. RestrictedRemoteServer loads these, but Empty does not.

- -VisibleAliases, -VisibleCmdlets, -VisibleFunctions, and -VisibleProviders: These comma-separated lists define which of the aliases, cmdlets, functions, and PSProviders you've loaded will actually be visible to the endpoint user. These enable you to load an entire module, but then only expose one or two commands, if desired.

**Note:** You can't use a custom endpoint alone to control which parameters a user will have access to. If you need that level of control, one option is to dive into .NET Framework programming, which does allow you to create a more fine-grained remote configuration. That's beyond the scope of this guide. You could also create a custom endpoint that only included proxy functions, another way of "wrapping" built-in commands and adding or removing parameters - but that's also beyond the scope of this guide.

Once you've created the configuration file, you're ready to register it. This is done with the Register-PSSessionConfiguration command, as shown in figure 3.4.

![image045.png](images/image045.png)

Figure 3.4: The Register-PSSessionConfiguration command

As you can see, there's a lot going on with this command. Some of the more interesting parameters include:

- -RunAsCredential: This lets you specify a credential that will be used to run all commands within the endpoint. Providing this credential enables users to connect and run commands that they normally wouldn't have permission to run; by limiting the available commands (via the session configuration file), you can restrict what users can do with this elevated privilege.
- -SecurityDescriptorSddl: This lets you specify who can connect to the endpoint. The specifier language is complex; consider using -ShowSecurityDescriptorUI instead, which shows a graphical dialog box to set the endpoint permissions.
- -StartupScript: This specifies a script to run each time the endpoint starts.

You can explore the other options on your own in the help file. Let's take a look at actually creating and using one of these custom endpoints. As shown in figure 3.5, we've created a new AD user account for SallyS of the Sales department. Sally, for some reason, needs to be able to list the users in our AD domain - but that's all she must be able to do. As-is, her account doesn't actually have permission to do so.

![image046.png](images/image046.png)

Figure 3.5: Creating a new AD user account to test

Figure 3.6 shows the creation of the new session configuration file, and the registration of the session. Notice that the session will auto-import the ActiveDirectory module, but only make the Get-ADUser cmdlet visible to Sally. We've specified a restricted remote session type, which will provide a few other key commands to Sally. We also disabled PowerShell's scripting language. When registering the configuration, we specified a "Run As" credential (we were prompted for the password), which is the account all commands will actually execute as.

![image047.png](images/image047.png)

Figure 3.6: Creating and registering the new endpoint

Because we used the -ShowSecurityDescriptorUI, we got a dialog box like the one shown in figure 3.7. This is an easier way of setting the permissions for who can use this new endpoint. Keep in mind that the endpoint will be running commands under a Domain Admin account, so we want to be very careful who we actually let in! Sally needs, at minimum, Execute and Read permission, which we've given her.

![image048.png](images/image048.png)

Figure 3.7: Setting the permissions on the endpoint

We then set a password for Sally and enabled her user account. Everything up to this point has been done on the DC01.AD2008R2.loc computer; figure 3.8 moves to that domain's Windows 7 client computer, where we logged in using Sally's account. As you can see, she was unable to enter the default session on the domain controller. But when she attempted to enter the special new session we set up just for her, she was successful. She was able to run Get-ADUser as well.

![image049.png](images/image049.png)

Figure 3.8: Testing the new endpoint by logging in as Sally

Figure 3.9 confirms that Sally has a very limited number of commands to play with. Some of these commands - like Get-Help and Exit-PSSession - are pretty crucial for using the endpoint. Others, like Select-Object, give Sally a minimal amount of non-destructive convenience for getting her command output to look like she needs. This command list (aside from Get-ADUser) is automatically set when you specify the "restricted remote" session type in the session configuration file.

![image050.png](images/image050.png)

Figure 3.9: Only eight commands, including the Get-ADUser one we added, are available within the endpoint.

In reality, it's unlikely that a Sales user like Sally would be running commands in the PowerShell console. More likely, she'd use some GUI-based application that ran the commands "behind the scenes." Either way, we've ensured that she has exactly the functionality she needs to do her job, and nothing more.

## Security Precautions with Custom Endpoints

When you create a custom session configuration file, as you've seen, you can set its language mode. The language mode determines what elements of the PowerShell scripting language are available in the endpoint - and the language mode can be a bit of a loophole. With the "Full" language mode, you get the entire scripting language, including script blocks. A script block is any executable hunk of PowerShell code contained within {curly brackets}. They're the loophole. Anytime you allow the use of script blocks, they can run any legal command - even if your endpoint used -VisibleCmdlets or -VisibleFunctions or another parameter to limit the commands in the endpoint.

In other words, if you register an endpoint that uses -VisibleCmdlets to only expose Get-ChildItem, but you create the endpoint's session configuration file to have the full language mode, then any script blocks inside the endpoint can use any command. Someone could run:

````
PS C:\> & { Import-Module ActiveDirectory; Get-ADUser -filter \* | Remove-ADObject }
````

Eek! This can be especially dangerous if you configured the endpoint to use a RunAs credential to run commands under elevated privileges. It's also somewhat easy to let this happen by mistake, because you set the language mode when you create the new session configuration file (New-PSSessionConfigurationFile), not when you register the session (Register-PSSessionConfiguration). So if you're using a session configuration file created by someone else, pop it open and confirm its language mode before you use it!

You can avoid this problem by setting the language mode to NoLanguage, which shuts off script blocks and the rest of the scripting language. Or, go for RestrictedLanguage, which blocks script blocks while still allowing some basic operators if you want users of the endpoint to be able to do basic filtering and comparisons.

Understand that this isn't a bug - the behavior we're describing here is by design. It can just be a problem if you don't know about it and understand what it's doing.

Note: Much thanks to fellow MVP Aleksandar Nikolic for helping me understand the logic of this loophole!



