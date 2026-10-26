# Criptografia Basica

La criptografia es la ciencia que estudia las tecnicas para proteger la informacion, transformando datos legibles en formatos incomprensibles para terceros no autorizados. Esta disciplina constituye el pilar fundamental de toda infraestructura de seguridad moderna, desde las comunicaciones TLS que protegen el trafico web hasta los mecanismos de autenticacion que verifican identidades digitales. Comprender sus principios es esencial para disenar sistemas seguros que resistan intentos de acceso no autorizado, manipulacion de datos y suplantacion de identidad.

## Criptografia Simetrica

La criptografia simmetrica utiliza una unica clave tanto para cifrar como para descifrar el texto. Esta misma clave debe compartirse entre las partes que desean comunicarse de forma segura, lo cual presenta el problema fundamental de distribucion de claves. Al ser computacionalmente eficiente, los algoritmos simetricos son la eleccion habitual para cifrar grandes volumenes de datos.

**AES (Advanced Encryption Standard)** es el estandar de cifrado simetrico mas utilizado en la actualidad. Opera con bloques de 128 bits y soporta claves de 128, 192 o 256 bits. Su seguridad se basa en sustituciones y permutaciones repetidas a lo largo de multiples rondas de procesamiento. AES-256 proporciona resistencia suficiente incluso contra ataques con computadoras cuanticas teoricas, por lo que se emplea en comunicaciones gubernamentales y financieras de alta sensibilidad.

**ChaCha20** es un cifrador de flujo dissenado por Daniel Bernstein que opera sobre bloques de 256 bits. A diferencia de AES, que trabaja con bloques fijos, ChaCha20 genera un flujo de bits pseudocaso que se combina con el texto plano mediante operacion XOR. Es especialmente eficiente en dispositivos sin hardware dedicado para AES, ya que su implementacion en software puro es rapidamente optimizable. ChaCha20-Poly1305 combina este cifrador con un sistema de autenticacion, proporcionando tanto confidencialidad como integridad en una sola operacion.

> En la practica, los algoritmos simmetricos se utilizan para cifrar la informacion una vez establecida un canal seguro, ya que el problema de compartir la clave secreta se resuelve mediante criptografia asimetrica.

## Criptografia Asimetrica

La criptografia asimetrica resuelve el problema de distribucion de claves mediante el uso de un par de claves mathematicalmente relacionadas: una clave publica y una clave privada. Lo que se cifra con una solo puede descifrarse con la otra, eliminando la necesidad de compartir secretos previamente.

**RSA (Rivest-Shamir-Adleman)** fue uno de los primeros sistemas de criptografia asimetrica y sigue siendo ampliamente utilizado. Su seguridad se basa en la dificultad computacional de factorizar productos de numeros primos grandes. Se recomienda usar claves de al menos 2048 bits, aunque 4096 bits proporciona una margen de seguridad mas amplio. RSA es comunmente utilizado para firmas digitales y para intercambio de claves, pero no para cifrar grandes volumenes de datos debido a su bajo rendimiento comparado con los algoritmos simmetricos.

**ECDSA (Elliptic Curve Digital Signature Algorithm)** se basa en la aritmetica de curvas elipticas, proporcionando la misma seguridad que RSA con claves significativamente mas cortas. Una curva eliptica de 256 bits ofrece una seguridad equivalente a RSA-3072, lo que se traduce en menor consumo de recursos y mejor rendimiento. ECDSA es el algoritmo preferido para firmas digitales en certificados TLS modernos, Ethereum y sistemas de autenticacion de alta seguridad.

> La relacion entre longitud de clave y seguridad no es lineal. Una clave ECDSA de 256 bits es tan segura como una RSA de 3072 bits, pero requiere significativamente menos poder computacional.

## Hashing

Las funciones de hash transforman datos de任意 longitud en un valor fijo denominado digest o resumen. A diferencia del cifrado, el hashing es una operacion unidireccional: es computacionalmente inviable reconstruir los datos originales a partir de su hash. Esta propiedad lo convierte en la herramienta ideal para verificar integridad y almacenar contraseñas de forma segura.

**SHA-256 (Secure Hash Algorithm 256)** produce un digest de 256 bits (32 bytes) y pertenece a la familia SHA-2. Es ampliamente utilizado en verificacion de integridad de archivos, blockchain, pruebas de trabajo y como componente de algoritmos de firma digital. Su resistencia a colisiones (dos entradas distintas que producen el mismo hash) lo hace fiable para aplicaciones de seguridad criticas. Sin embargo, no debe utilizarse directamente para almacenar contraseñas ya que es demasiado rapido, facilitando ataques de fuerza bruta.

**bcrypt** es una funcion de hashing dissenada especificamente para contraseñas. Incorpora un salt aleatorio y un factor de costo configurable que controla la complejidad computacional del proceso. A diferencia de SHA-256, bcrypt es intencionalmente lento: cada incremento en el factor de costo duplica el tiempo de procesamiento. Esto hace que los ataques de fuerza bruta sean prohibitivamente costosos. El salt evita que se utilicen tablas de arco iris para descifrar contraseñas hashes.

## Firma Digital

La firma digital combina criptografia asimetrica y hashing para proporcionar autenticacion e integridad. El proceso consiste en calcular el hash del documento original y luego cifrar ese hash con la clave privada del firmante. Cualquier persona con la clave publica puede verificar la firma descifrando el hash y comparandolo con una recalculacion independiente.

La firma digital garantiza tres propiedades fundamentales: autenticacion (verifica que el emisor posee la clave privada), integridad (cualquier modificacion del documento invalida la firma) y no repudio (el firmante no puede negar haber firmado, dado que solo el poseedor de la clave privada podria haber generado la firma). Estas propiedades son esenciales para certificados digitales, contratos electronicos y la validacion de actualizaciones de software.

## Cifrado vs Hashing

La diferencia fundamental entre cifrado y hashing radica en la reversibilidad. El cifrado es una operacion reversible: con la clave correcta, el texto cifrado se transforma de vuelta en texto plano. El hashing es una operacion unidireccional: no existe proceso computacionalmente viable para recuperar los datos originales a partir del digest.

En la practica, ambos se complementan. El cifrado protege la confidencialidad de datos en transito o en reposo, mientras que el hashing verifica integridad y almacena credenciales. Un sistema seguro tipicamente utiliza ambos: cifrado para comunicaciones, hashing para verificacion de integridad y almacenamiento de contraseñas. Comprender esta distincion es crucial para elegir la herramienta adecuada para cada requisito de seguridad.

---

