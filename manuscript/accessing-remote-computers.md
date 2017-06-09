# Acceso a equipos remotos

Principalmente existen dos escenarios al acceder una computadora remota. La diferencia entre estos escenarios radica especialmente en la respuesta a una pregunta: ¿Puede WinRM identificar y autenticar la máquina remota?

Obviamente, la máquina remota necesita saber quién es usted, porque estará ejecutando comandos en su nombre. Pero usted necesita saber quién es, también. Esta autenticación mutua, es un paso de seguridad importante. Significa que cuando escribe SERVER2, se está conectando realmente con el SERVER2 real, y no con alguna máquina haciéndose pasar por SERVER2. Mucha gente ha publicado artículos de blog sobre cómo deshabilitar las verificaciones de autenticación. Hacerlo, hace que Remoting " funcione" y se deshaga de los molestos mensajes de error, pero abre brechas de seguridad y hace posible que alguien "secuestre" o "falsifique" su conexión y potencialmente capture información confidencial como sus credenciales
.

**Precaución:** Tenga en cuenta que Remoting implica delegar una credencial en el equipo remoto. Usted está haciendo algo más que simplemente enviar un nombre de usuario y una contraseña \(que en realidad no ocurre todo el tiempo\). Está dando a la máquina remota la capacidad de ejecutar las tareas como si estuviera allí ejecutándolas usted mismo. Un impostor podría hacer mucho daño con ese poder. Es por eso que Remoting se enfoca en la autenticación mutua, para que los impostores no tengan esa oportunidad.

En los escenarios de Remoting más sencillos, usted se conecta a una máquina que está en el mismo dominio de AD utilizando su nombre de equipo real, tal como está registrado en AD. AD maneja la autenticación mutua y todo funciona de maravilla. Pero las cosas se pueden poner un poco más difíciles en otros escenarios:

* Conectar a una máquina en otro dominio
* Conectar a una máquina que no está en un dominio en absoluto
* Conectarse a través de un alias de DNS, o a través de una dirección IP, en lugar de a través del nombre de equipo real de la máquina como está registrado con AD

En estos casos, AD no puede hacer la autenticación mutua, por lo que tendrá que hacerlo usted mismo. En este punto tiene dos opciones:

* Configurar la máquina remota para aceptar conexiones HTTPS \(en lugar de HTTP\) y equiparla con un certificado SSL. El Certificado SSL debe ser emitido por una Autoridad de Certificación \(CA\) en la que confíe la máquina. Esto permite que el certificado SSL proporcione la autenticación mutua que WinRM usara luego.
* O, agregar el nombre de la máquina remota \(lo que esté especificando, ya sea un nombre de equipo real, una dirección IP o un alias CNAME\) a la lista de WinRM TrustedHosts de su equipo local. Tenga en cuenta que esto básicamente inhabilita la autenticación mutua, ya que permite a WinRM conectarse con ese identificador \(nombre, dirección IP o lo que sea\) sin utilizar la autenticación mutua. Esto abre la posibilidad para que una máquina pretenda ser la que usted desea, así que es mejor que tenga la debida precaución.

En ambos casos, también debe especificar un parámetro -Credential en el comando Remoting, aunque sólo esté especificando la misma credencial que está utilizando para ejecutar PowerShell. Cubriremos ambos casos en las siguientes dos secciones.

**Nota:** A lo largo de esta guía, usaremos "Comando Remoting" para referirnos genéricamente a cualquier comando que implique la creación de una conexión Remoting. Estos incluyen \(pero no se limitan a\) New-PSSession, Enter-PSSession, Invoke-Command, y así sucesivamente.

## Configuración de un HTTPS Listener

Esta es una de las cosas más complejas que puede hacer con Remoting, e implicará ejecutar una gran cantidad de utilitarios externos. Lo siento - es sólo que así se hace- En este momento no parece haber una manera fácil de hacer esto totalmente desde PowerShell, o al menos no la hemos encontrado. Algunas cosas, podrían hacerse a través de PowerShell, pero como resulta más fácil hacerlo de otra forma, así lo he hecho.

El primer paso es identificar el nombre del host que la gente utilizará para acceder a su servidor. Esto es muy, muy importante, y no es necesariamente lo mismo que el nombre de equipo real del servidor. Por ejemplo, la gente que accede a "www.ad2008r2.loc" podría estar golpeando un servidor llamado "DC01", pero el certificado SSL que creará debe ser emitido para el nombre de host "www.ad2008r2.loc" porque eso es lo que la gente estará escribiendo Por lo tanto, el nombre del certificado debe coincidir con el nombre que la gente va a escribir para llegar a la máquina - incluso si es diferente de su verdadero nombre de equipo. ¿Lo tiene?

**Nota:** Nota: Parte de la configuración de un listener de HTTPS es obtener un certificado SSL. Utilizaré una Autoridad de Certificación \(CA\) pública llamada DigiCert.com. También puede usar una PKI interna, si su organización tiene una. No recomiendo usar MakeCert.exe, ya que los equipos que intentan conectarse no pueden confiar implícitamente en dicho certificado. Me doy cuenta de que cada blog en el universo le dice que use MakeCert.exe para crear un certificado auto-firmado local. Sí, es fácil, pero está mal. Usarlo requiere que apague la mayor parte de la seguridad de WinRM, así que ¿por qué molestarse con SSL si planea apagar la mayoría de sus características de seguridad?

También necesita asegurarse de conocer el nombre completo usado para conectar con una computadora. Si la gente tiene que escribir "dc01.ad2008r2.loc", entonces eso es lo que debe aparecer en el certificado. Si simplemente necesita digitar "dca", y saber que un DNS puede resolver eso a una dirección IP, entonces "dca" es lo que debe llevar el certificado. Estamos creando un certificado que solo dice "dca" y debemos asegurarnos que nuestros equipos puedan resolver eso a una dirección IP.

#### Creación de una solicitud de certificado

A diferencia de IIS, PowerShell no ofrece una forma amigable y gráfica de crear una Solicitud de Certificado (de hecho no ofrece ninguna). Entonces, vaya a [http://DigiCert.com/util](http://DigiCert.com/util) y descargue su versión gratuita del “Utilitario para certificados”. La Figura 2.1 muestra el utilitario. Tenga en cuenta el mensaje de advertencia.

![image008.png](images/image008.png)

Figura 2.1: Ejecutando DigiCertUtil.exe

Sólo tiene que preocuparse por la advertencia si planea adquirir su certificado de la CA de DigiCert. Haga clic en el botón Repair para instalar los certificados intermedios en su computadora, permitiendo que su certificado sea confiable y se pueda utilizar. La figura 2.2 muestra el resultado de hacerlo. Una vez más, si planea llevar la Solicitud de Certificado \(CSR\) eventual a una CA diferente, no se preocupe por el botón Repair o por el mensaje de advertencia

**Nota** También puede abrir una consola MMC en blanco y agregar el complemento "Certificados" de Windows. Cuando se le solicite, agregue la “cuenta de equipo” para el equipo local. A continuación, haga clic con el botón derecho en la carpeta "Personal" y seleccione Todas las tareas para encontrar la opción para crear una nueva solicitud de certificado.  
:  
![image009.png](images/image009.png)

Figura 2.2: Después de agregar los certificados intermedios de DigiCert

Haga clic en " Create CSR". Como se muestra en la figura 2.3, complete la información sobre su organización. Esto tiene que ser exacto: El "nombre común" es exactamente lo que la gente escribirá para acceder al equipo en el que se instalará este certificado SSL. Podría ser simplemente "dca", en nuestro caso, o "dc01.ad20082.loc" si se necesita un nombre completo, y así sucesivamente. El nombre de su empresa también debe ser preciso: la mayoría de las CA verificarán esta información.

![image010.png](images/image010.png)

Figura 2.3: Diligenciar el CSR

Por lo general, se guarda la CSR en un archivo de texto, como se muestra en la figura 2.4. También puede copiarlo en el Portapapeles. Cuando vaya a su CA, asegúrese de que está solicitando un certificado SSL \("Servidor Web", en algunos casos\). Un certificado de correo electrónico u otro tipo no funcionará.

![image011.png](images/image011.png)

Figura 2.4: Guardar el CSR en un archivo de texto

A continuación, lleve esa CSR a su CA y solicite su certificado. Verá algo como la figura 2.5 si está utilizando DigiCert. Obviamente será diferente con otra CA, con una PKI interna. Tenga en cuenta que con la mayoría de las CA comerciales tendrá que seleccionar el tipo de servidor Web que está utilizando \(IIS o el que corresponda\).

**Nota**: El uso del utilitario MakeCert.exe en el SDK de Windows generará un certificado local en el que solo su máquina confiará. Esto no es útil. Mucha gente le dirá que haga esto en varias publicaciones o blogs, porque es rápido y fácil. También le dirán que deshabilite algunas  comprobaciones de seguridad para que el certificado inherentemente inútil funcione. Es una pérdida de tiempo. Usted estará utilizando cifrado, pero no tendrá la seguridad de que la máquina remota es a la que tenía la intención de conectarse. Si alguien está secuestrando su información, ¿a quién le importa si se cifró antes de enviarla a ellos?

![image012.png](images/image012.png)

Figura 2.5: Carga del CSR en una CA

**Precaución**: Observe el mensaje de advertencia en la figura 2.5. Mi CSR necesita ser generado con una clave de 2048 bits. La utilidad de DigiCert ofrece eso o 1024 bits. Muchas CA tendrán un requisito de bit-alto. Asegúrese de que su RSE cumple con lo que necesita. Observe también que se trata de un certificado de servidor Web lo que estamos solicitando. Como escribimos anteriormente, es el único tipo de certificado que funcionará.

Eventualmente, la CA emitirá su certificado. La Figura 2.6 muestra el sitio a dónde fuimos para descargarlo. Elegimos descargar todos los certificados. Queríamos asegurarnos de tener una copia del certificado raíz de la CA, en caso de que necesitáramos configurar otra máquina para confiar en esa raíz.

**Sugerencia**: El truco con los certificados digitales es que la máquina que los utiliza y las máquinas a las que se presentarán, deben confiar en la entidad emisora de certificados que emitió el mismo. Es por eso que descarga el certificado raíz de la CA, para que pueda instalarlo en las máquinas que necesitan confiar en dicha CA. En un entorno grande, esto se puede hacer a través de una directiva de grupo, si se quisiera.

![image013.png](images/image013.png)

Figura 2.6: Descarga del certificado emitido

Asegúrese de hacer una copia de seguridad de los archivos de certificados. Aunque la mayoría de las CA las publicarían de nuevo de ser necesario, es mucho más fácil tener una copia de seguridad.

#### Instalación del certificado

No intente hacer doble clic en el archivo de certificado para instalarlo. Si lo hace, lo instalará en el almacén de certificados de su cuenta de usuario. Lo necesita en el almacén de certificados de su computadora. Para instalar el certificado, abra una nueva consola de administración de Microsoft (mmc.exe), seleccione Agregar o quitar complementos y agregue el complemento Certificados, como se muestra en la figura 2.7.

![image014.png](images/image014.png)

Figura 2.7: Agregar el complemento Certificados a la MMC

Como se muestra en la figura 2.8, establezca el complemento en la cuenta de equipo.

![image015.png](images/image015.png)

Figura 2.8: Establecer el complemento Certificados en la cuenta de equipo

A continuación, como se muestra en la figura 2.9, establezca el equipo local. Por supuesto, si está instalando un certificado en una computadora remota, establezca esa computadora en su lugar. Esta es una buena forma de instalar un certificado en un ambiente Windows sin GUI como en un Server Core, por ejemplo.

Nota: Quisiéramos poder mostrarle una forma de hacer todo esto desde PowerShell. No pudimos encontrar una que no implicara un montón de pasos más además de complejos. Dado que esto no es algo que tendrá que hacer a menudo o automatizarlo, la GUI es más fácil y debería ser suficiente.

![image016.png](images/image016.png)

Figura 2.9: Establecer el complemento Certificados en el equipo local

Con el complemento cargado, como se muestra en la figura 2.10, haga clic con el botón derecho en el almacén "Personal" y seleccione "Import".

![image017.png](images/image017.png)

Figura 2.10: Inicio del proceso de importación en el almacén Personal

Como se muestra en la figura 2.11, vaya al archivo de certificado que descargó de su CA. A continuación, haga clic en Next.

**Precaución**: Si ha descargado varios certificados, tal vez los certificados raíz de la CA junto con el certificado, asegúrese de importar el certificado SSL que se le entregó. Si hay alguna confusión, PARE. Vuelva a su CA y descargue sólo su certificado, para que sepa cuál importar. No experimente, necesita realizar bien esto a la primera vez.

![image018.png](images/image018.png)

Figura 2.11: Selección del archivo de certificado SSL recién publicado

Como se muestra en la figura 2.12, asegúrese de que el certificado se ubicará en el almacén Personal.

![image019.png](images/image019.png)

Figura 2.12: Asegúrese de ubicar el certificado en el almacén Personal, que debe estar preseleccionado.

Como se muestra en la figura 2.13, haga doble clic en el certificado para abrirlo. O bien, haga clic con el botón derecho y seleccione Abrir. No seleccione Propiedades - no le proporcionará la información que necesita-.

![image020.png](images/image020.png)

Figura 2.13: Haga doble clic en el certificado o haga clic con el botón derecho del ratón y seleccione Open

Finalmente, como se muestra en la figura 2.14, seleccione la huella digital del certificado. Deberá anotar esto o copiarlo en el Portapapeles. Así WinRM identificará el certificado que desea utilizar.

**Nota**: Es posible listar su certificado en la unidad CERT: de PowerShell, lo que hará que la huella digital sea más fácil de copiar en el Portapapeles. En PowerShell, ejecute Dir CERT:\LocalMachine\My. Asegúrese que selecciona el certificado correcto. Si no se muestra toda la huella digital, ejecute Dir CERT:\LocalMachine\My \| FL \* en su lugar.

![image021.png](images/image021.png)

Figure 2.14: Obtaining the certificate's thumbprint

#### Setting up the HTTPS Listener

These next steps will be accomplished in the Cmd.exe shell, not in PowerShell. The command-line utility's syntax requires significant tweaking and escaping in PowerShell, and it's a lot easier to type and understand in the older Cmd.exe shell \(which is where the utility has to run anyway; running it in PowerShell would just launch Cmd.exe behind the scenes\).

As shown in figure 2.15, run the following command:

![image022.png](images/image022.png)

Figure 2.15: Setting up the HTTPS WinRM listener

```
Winrm create winrm/config/Listener?Address=\*+Transport=HTTPS @{Hostname="xxx";CertificateThumbprint="yyy"}
```

There are two or three pieces of information you'll need to place into this command:

* In place of \*, you can put an individual IP address. Using \* will have the listener listen to all local IP addresses.
* In place of xxx, put the exact computer name that the certificate was issued to. If that includes a domain name \(such as dc01.ad2008r2.loc\), put that. Whatever's in the certificate must go here, or you'll get a CN mismatch error. Our certificate was issued to "dca," so I put "dca."
* In place of yyy, put the exact certificate thumbprint that you copied earlier. It's okay if this contains spaces.

That's all you should need to do in order to get the listener working.

**Note:** We had the Windows Firewall disabled on this server, so we didn't need to create an exception. The exception isn't created automatically, so if you have any firewall enabled on your computer, you'll need to manually create the exception for port 5986.

You can also run an equivalent PowerShell command to accomplish this task:

```
New-WSManInstance winrm/config/Listener -SelectorSet @{Address='\*';
Transport='HTTPS'} -ValueSet @{HostName='xxx';CertificateThumbprint='yyy'}
```

In that example, "xxx" and "yyy" get replaced just as they did in the previous example.

#### Testing the HTTPS Listener

I tested this from the standalone C3925954503 computer, attempting to reach the DCA domain controller in COMPANY.loc. I configured C3925954503 with a HOSTS file, so that it could resolve the hostname DCA to the correct IP address without needing DNS. I was sure to run:

```
Ipconfig /flushdns
```

This ensured that the HOSTS file was read into the DNS name cache. The results are in figure 2.16. Note that I can't access DCA by using its IP address directly, because the SSL certificate doesn't contain an IP address. The SSL certificate was issued to "dca," so we need to be able to access the computer by typing "dca" as the computer name. Using the HOSTS file will let Windows resolve that to an IP address.

**Note:** Remember, there are two things going on here: Windows needs to be able to resolve the name to an IP address, which is what the HOSTS file accomplishes, in order to make a physical connection. But WinRM needs mutual authentication, which means whatever we typed into the -ComputerName parameter needs to match what's in the SSL certificate. That's why we couldn't just provide an IP address to the command - it would have worked for the connection, but not the authentication.

![image023.png](images/image023.png)

Figure 2.16: Testing the HTTPS listener

We started with this:

```
Enter-PSSession -computerName DCA
```

It didn't work - which I expected. Then we tried this:

```
Enter-PSSession -computerName DCA -credential COMPANY\Administrator
```

We provided a valid password for the Administrator account, but as expected the command didn't work. Finally:

```
Enter-PSSession -computerName DCA -credential COMPANY\Administrator -UseSSL
```

Again providing a valid password, we were rewarded with the remote prompt we expected. It worked! This fulfills the two conditions we specified earlier: We're using an HTTPS-secured connection _and_ providing a credential. Both conditions are required because the computer isn't in my domain \(since in this case the source computer isn't even in a domain\). As a refresher, figure 2.17 shows, in green, the connection we created and used.

![image024.png](images/image024.png)

Figure 2.17: The connection used for the HTTPS listener test

#### Modifications

There are two modifications you can make to a connection, whether using Invoke-Command, Enter-PSSession, or some other Remoting command, which relate to HTTPS listeners. These are created as part of a session option object.

* -SkipCACheck causes WinRM to not worry about whether the SSL certificate was issued by a trusted CA or not. However, untrusted CAs may in fact be untrustworthy! A poor CA might issue a certificate to a bogus computer, leading you to believe you're connecting to the right machine when in fact you're connecting to an imposter. This is risky, so use it with caution.
* -SkipCNCheck causes WinRM to not worry about whether the SSL certificate on the remote machine was actually issued for that machine or not. Again, this is a great way to find yourself connected to an imposter. Half the point of SSL is mutual authentication, and this parameter disables that half.

Using either or both of these options will still enable SSL encryption on the connection - but you'll have defeated the other essential purpose of SSL, which is mutual authentication by means of a trusted intermediate authority.

To create and use a session object that includes both of these parameters:

```
$option = New-PSSessionOption -SkipCACheck -SkipCNCheck
Enter-PSSession -computerName DCA -sessionOption $option
        -credential COMPANY\Administrator -useSSL
```

**Caution:** Yes, this is an easy way to make annoying error messages go away. But those errors are trying to warn you of a potential problem and protect you from potential security risks that are very real, and which are very much in use by modern attackers.

## Certificate Authentication

Once you have an HTTPS listener set up, you have the option of authenticating with Certificates. This allows you to connect to remote computers, even those in an untrusted domain or workgroup, without requiring either user input or a saved password. This may come in handy when scheduling a task to run a PowerShell script, for example.

In Certificate Authentication, the client holds a certificate with a private key, and the remote computer maps that certificate's public key to a local Windows account. WinRM requires a certificate which has "Client Authentication \(1.3.6.1.5.5.7.3.2\)" listed in the Enhanced Key Usage attribute, and which has a User Principal Name listed in the Subject Alternative Name attribute. If you're using a Microsoft Enterprise Certification Authority, the "User" certificate template meets these requirements.

#### Obtaining a certificate for client authentication

These instructions assume that you have a Microsoft Enterprise CA. If you are using a different method of certificate enrollment, follow the instructions provided by your vendor or CA administrator.

On your client computer, perform the following steps:

* Run certmgr.msc to open the "Certificates - Current User" console.
* Right click on the "Personal" node, and select All Tasks -&gt; Request New Certificate& 
* In the Certificate Enrollment dialog, click Next. Highlight "Active Directory Enrollment Policy", and click Next again. Select the User template, and click Enroll.

![image025.png](images/image025.png)

Figure 2.18: Requesting a User certificate.

After the Enrollment process is complete and you're back at the Certificates console, you should now see the new certificate in the Personal\Certificates folder:

![image026.png](images/image026.png)

Figure 2.19: The user's installed Client Authentication certificate.

Before closing the Certificates console, right-click on the new certificate, and choose All Tasks -&gt; Export. In the screens that follow, choose "do not export the private key", and save the certificate to a file on disk. Copy the exported certificate to the remote computer, for use in the next steps.

#### Configuring the remote computer to allow Certificate Authentication

On the remote computer, run the PowerShell console as Administrator, and enter the following command to enable Certificate authentication:

```
Set-Item -Path WSMan:\localhost\Service\Auth\Certificate -Value $true
```

#### Importing the client's certificate on the remote computer

The client's certificate must be added to the machine "Trusted People" certificate store. To do this, perform the following steps to open the "Certificates \(Local Computer\)" console:

* Run "mmc".
* From the File menu, choose "Add/Remove Snap-in."
* Highlight "Certificates", and click the Add button.
* Select the "Computer Account" option, and click Next.
* Select "Local Computer", and click Finish, then click OK.

**Note:** This is the same process you followed in the "Installing the Certificate" section under Setting up and HTTPS Listener. Refer to figures 2.7, 2.8 and 2.9 if needed.

In the Certificates \(Local Computer\) console, right-click the "Trusted People" store, and select All Tasks -&gt; Import.

![image027.png](images/image027.png)

Figure 2.20: Starting the Certificate Import process.

Click Next, and Browse to the location where you copied the user's certificate file.

![image028.png](images/image028.png)

Figure 2.21: Selecting the user's certificate.

Ensure that the certificate is placed into the Trusted People store:

![image029.png](images/image029.png)

Figure 2.22: Placing the certificate into the Trusted People store.

#### Creating a Client Certificate mapping on the remote computer

Open a PowerShell console as Administrator on the remote computer. For this next step, you will require the Certificate Thumbprint of the CA that issued the client's certificate. You should be able to find this by issuing one of the following two commands \(depending on whether the CA's certificate is located in the "Trusted Root Certification Authorities" or the "Intermediate Certification Authorities" store\):

```
Get-ChildItem -Path cert:\LocalMachine\Root  
Get-ChildItem -Path cert:\LocalMachine\CA
```

![image030.png](images/image030.png)

Figure 2.23: Obtaining the CA certificate thumbprint.

Once you have the thumbprint, issue the following command to create the certificate mapping:

```
New-Item -Path WSMan:\localhost\ClientCertificate -Credential (Get-Credential) -Subject <userPrincipalName> -URI \* -Issuer <CA Thumbprint> -Force
```

When prompted for credentials, enter the username and password of a local account with Administrator rights.

**Note:** It is not possible to specify the credentials of a domain account for certificate mapping, even if the remote computer is a member of a domain. You must use a local account, and the account must be a member of the Administrators group.

![image031.png](images/image031.png)

Figure 2.24: Setting up the client certificate mapping.

#### Connecting to the remote computer using Certificate Authentication

Now, you should be all set to authenticate to the remote computer using your certificate. For this step, you will need the thumbprint of the client authentication certificate. To obtain this, you can run the following command on the client computer:

```
Get-ChildItem -Path Cert:\CurrentUser\My
```

Once you have this thumbprint, you can authenticate to the remote computer by using either the Invoke-Command or New-PSSession cmdlets with the -CertificateThumbprint parameter, as shown in figure 2.25.

**Note:** The Enter-PSSession cmdlet does not appear to work with the -CertificateThumbprint parameter. If you want to enter an interactive remoting session with certificate authentication, use New-PSSession first, and then Enter-PSSession.

**Note:** The -UseSSL switch is implied when you use -CertificateThumbprint in either of these commands. Even if you don't type -UseSSL, you're still connecting to the remote computer over HTTPS \(port 5986, by default, on Windows 7 / 2008 R2 or later\). Figure 2.26 demonstrates this.

![image032.png](images/image032.png)

Figure 2.25: Using a certificate to authenticate with PowerShell Remoting.

![image033.png](images/image033.png)

Figure 2.26: Demonstrating that the connection is over SSL port 5986, even without the -UseSSL switch.

## Modifying the TrustedHosts List

As I mentioned earlier, using SSL is only one option for connecting to a computer for which mutual authentication isn't possible. The other option is to selectively disable the need for mutual authentication by providing your computer with a list of "trusted hosts." In other words, you're telling your computer, "If I try to access SERVER1 \[for example\], don't bother mutually authenticating. I know that SERVER1 can't possibly be spoofed or impersonated, so I'm taking that burden off of your shoulders."

Figure 2.27 illustrates the connection we'll be attempting.

![image034.png](images/image034.png)

Figure 2.27: The TrustedHosts connection test

Beginning on CLIENTA, with a completely default Remoting configuration, we'll attempt to connect to C3925954503, which also has a completely default Remoting configuration. Figure 2.28 shows the result. Note that I'm connecting via IP address, rather than hostname; our client has no way of resolving the computer's name to an IP address, and for this test we'd rather not modify my local HOSTS file.

![image035.png](images/image035.png)

Figure 2.28: Attempting to connect to the remote computer

This is what we expected: The error message is clear that we can't use an IP address \(or a host name for a non-domain computer, although the error doesn't say so\) unless we either use HTTPS and a credential, or add the computer to my TrustedHosts list and use a credential. We'll choose the latter this time; figure 2.29 shows the command we need to run. If we'd wanted to connect via the computer's name \(C3925954503\) instead of its IP address, we'd have added that computer name to the TrustedHosts list \(It'd be our responsibility to ensure my computer could somehow resolve that computer name to an IP address to make the physical connection\).

![image036.png](images/image036.png)

Figure 2.29: Adding the remote machine to our TrustedHosts list

This is another case where many blogs will advise just putting "\*" in the TrustedHosts list. Really? There's no chance any computer, ever, anywhere, could be impersonated or spoofed? We prefer adding a limited, controlled set of host names or IP addresses. Use a comma-separated list; it's okay to use wildcards along with some other characters \(like a domain name, such as \*.COMPANY.loc\), to allow a wide, but not unlimited, range of hosts. Figure 2.30 shows the successful connection.

**Tip:** Use the -Concatenate parameter of Set-Item to add your new value to any existing ones, rather than overwriting them.

![image037.png](images/image037.png)

Figure 2.30: Connecting to the remote computer

Managing the TrustedHosts list is probably the easiest way to connect to a computer that can't offer mutual authentication, provided you're absolutely certain that spoofing or impersonation isn't a possibility. On an intranet, for example, where you already exercise good security practices, impersonation may be a remote chance, and you can add an IP address range or host name range using wildcards.

## Connecting Across Domains

Figure 2.31 illustrates the next connection we'll try to make, which is between two computers in different, trusted and trusting, forests.

![image038.png](images/image038.png)

Figure 2.31: Connection for the cross-domain test

Our first test is in figure 2.32. Notice that we're creating a reusable credential in the variable $cred, so that we don't keep having to re-type the password as we try this. However, the results of the Remoting test still aren't successful.

![image039.png](images/image039.png)

Figure 2.32: Attempting to connect to the remote computer

The problem? We're using a CNAME alias \(MEMBER1\), not the computer's real host name \(C2108222963\). While WinRM can use a CNAME to resolve a name to an IP address for the physical connection, it can't use the CNAME alias to look the computer up in AD, because AD doesn't use the CNAME record \(even in an AD-integrated DNS zone\). As shown in figure 2.33, the solution is to use the computer's real host name.

![image040.png](images/image040.png)

Figure 2.33: Successfully connecting across domains

What if you _need_ to use an IP address or CNAME alias to connect? Then you'll have to fall back to the TrustedHosts list or an HTTPS listener, exactly as if you were connecting to a non-domain computer. Essentially, if you can't use the computer's real host name, as listed in AD, then you can't rely on the domain to shortcut the whole authentication process.

## Administrators from Other Domains

There's a quirk in Windows that tends to strip the Administrator account token for administrator accounts coming in from other domains, meaning they end up running under standard user privileges - which often isn't sufficient. In the target domain, you need to change that behavior.

To do so, run this on the target computer \(type this all in one line and then hit Enter\):

```
New-ItemProperty -Name LocalAccountTokenFilterPolicy
-Path HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\
Policies\System -PropertyType Dword -Value 1
```

That should fix the problem. Note that this does disable User Account Control \(UAC\) on the machine where you ran it, so make sure that's okay with you before doing so.

## The Second Hop

One default limitation with Remoting is often referred to as the second hop. Figure 2.25 illustrates the basic problem: You can make a Remoting connection from one host to another \(the green line\), but going from that second host to a third \(the red line\) is simply disallowed. This "second hop" doesn't work because, by default, Remoting can't delegate your credential a second time. This is even a problem if you make the first hop and subsequently try to access any network resource that requires authentication. For example, if you remote into another computer, and then ask that computer to access something on an authenticated file share, the operation fails.

### The CredSSP Solution

The following configuration changes are needed to enable the second hop:

**Note:** This only works on Windows Vista, Windows Server 2008, and later versions of Windows. It won't work on Windows XP or Windows Server 2003 or earlier versions.

* CredSSP must be enabled on your originating computer and the intermediate server you connect to. In PowerShell, on your originating computer, run:

```
Set-Item WSMAN:\localhost\client\auth\credssp -value $true
```

* On your intermediate server\(s\), you make a similar change to the above, but in a different section of the configuration:

```
Set-Item WSMAN:\localhost\service\auth\credssp -value $true
```

* Your domain policy must permit delegation of fresh credentials. In a Group Policy object \(GPO\), this is found in Computer Configuration &gt; Policies &gt; Administrative Templates &gt; System &gt; Credential Delegation &gt; Allow Delegation of Fresh Credentials. You must provide the names of the machines to which credentials may be delegated, or specify a wildcard like "\*.ad2008r2.loc" to allow an entire domain. Be sure to allow time for the updated GPO to apply, or run Gpupdate on the originating computer \(or reboot it\).

**Note:** Once again, the name you provide here is important. Whatever you'll actually be typing for the -computerName parameter is what must appear here. This makes it really tough to delegate credentials to, say, IP addresses, without just adding "\*" as an allowed delegate. Adding "\*," of course, means you can delegate to ANY computer, which is potentially dangerous, as it makes it easier for an attacker to impersonate a machine and get hold of your super-privileged Domain Admin account!

* When running a Remoting command, you must specify the "-Authentication CredSSP" parameter. You must also use the -Credential parameter and supply a valid DOMAIN\Username \(you'll be prompted for the password\) - even if it's the same username that you used to open PowerShell in the first place.

After setting the above, we were able to use Enter-PSSession to go from our domain controller to my member server, and then use Invoke-Command to run a command on a client computer - the connection illustrated in figure 2.34.

![image041.png](images/image041.png)

Figure 2.34: The connections for the second-hop test

Seem tedious and time-consuming to make all of those changes? There's a faster way. On the originating computer, run this:

```
Enable-WSManCredSSP -Role Client -Delegate name
```

Where "name" is the name of the computers that you plan to remote to next. This can be a wildcard, like \*, or a partial wildcard, like \*.AD2008R2.loc. Then, on the intermediate computer \(the one to which you will delegate your credentials\), run this:

```
Enable-WSManCredSSP -Role Server
```

Between them, these two commands will accomplish almost all of the configuration points we listed earlier. The only exception is that they will modify your local policy to permit fresh credential delegation, rather than modifying domain policy via a GPO. You can choose to modify the domain policy yourself, using the GPMC, to make that particular setting more universal.

### The Kerberos Solution

CredSSP isn't considered the safest protocol in the world \(see https://msdn.microsoft.com/en-us/library/cc226796.aspx\). Credentials _are_ transmitted, for example, and in clear text too, which is a problem. Fortunately, _within a domain, _there's another way to enable multi-hop Remoting, using the native Kerberos protocol, which does _not_ transmit credentials. Specifically, it's called Resource-Based Kerberos constraint delegation, and Microsoft PFE Ashley McGlone \(@goateePFE\) [wrote about it](https://blogs.technet.microsoft.com/ashleymcglone/2016/08/30/powershell-remoting-kerberos-double-hop-solved-securely/). 

This basic technique works since Windows Server 2003, so it should cover any situations you need. The idea here is that one machine can be allowed to delegate credentials _specific services on another machine. _Windows Server 2012 simplified the design of this previously undocumented, complex technique, and so we'll focus on that. So, every machine involved needs to have Windows Server 2012 or later, including at least one Win2012 domain controller in the domain. You'll also need a late-model Windows computer with the RSAT installed \(I used Windows 10\). You'll know you've got the run version if you can run:

```
Import-Module ActiveDirectory
Get-Command -ParameterName PrincipalsAllowedToDelegateToAccount
```

And get some results back. If you get nothing, you've got an older version of the RSAT - you need a newer one, which will likely require a newer version of Windows on your client. So, let's say we're on ClientA, we want to connect to ServerB, and have it delegate a credential across a second hop to ServerC.

```
$ClientA = $env:COMPUTERNAME
$ServerB = Get-ADComputer -Identity ServerB
$ServerC = Get-ADComputer -Identity ServerC

Set-ADComputer -Identity $ServerC -PrincipalsAllowedToDelegateToAccount $ServerB
```

This allows ServerC to accept a delegated credential from ServerB. That ability it an attribute of ServerC, if you're paying attention, meaning the _computer at the end of the second hop_ is what you modify, so that it can receive a credential from the middleman. Additionally, if you've already attempted a second-hop before setting this up, you need to wit about 15 minutes for Active Directory's "bad computer cache" to expire and allow all this to actually work. You could also just reboot ServerB, if you're in a lab or something and that's an option.

The -PrincipalsAllowedToDelegateToAccount can also be an array, as in @\($ServerB,$ServerZ,$ServerX\), etc, allowing multiple origins to delegate a credential to the machine account you're updating. And you can make this work across trust boundaries, too - see Ashley's original article for the technique.

