---
layout: default
title: Djb2
parent: Funciones de hash
grand_parent: Criptografia
---

# Djb2
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Funcionamiento

1. Se define 5381 como el valor inicial del hash
2. Por cada caracter del texto a hashear se realiza una operacion de desplazamiento y suma sobre el valor actual de hash
3. Multiplica el hash actual por 33 y luego suma el valor ASCII del caracter actual del texto

La multplicacion por 33 y la suma del hash actual se sustitute por ```(hash  << 5) + hash```, ambas operaciones son lo mismo.

### ¿Por que 33 y 5381?

La principal razon de por que se usa ```33``` y ```5381``` es para evitar colisiones y mejorar el afecto avalancha en la funcion de hash, esto ultimo hace referencia a que si la entrada de la funcion de hash cambia lo mas minimo, la salida cambia completamente o en su mayoria cambia, como se observa en esta imagen para SHA-1, cambiar un solo bit hace que la salida cambie totalmente

![](https://upload.wikimedia.org/wikipedia/commons/1/15/Avalanche_effect.svg)

Asi mismo, estas "contantes magicas" no se seleccionan al azar, en el caso de djb2, el numero 33 obtenia una buena distribucion uniforme para distribuir de mejor manera los valores del hash.

Otra razon es por la velocidad de multiplicar por 33 que es equivalente a ```hash * (32 + 1)``` que se puede rescribir como  ```(hash  << 5) + hash```, al desplazar a la izquierda  5 posiciones los bits del hash es equivalente a multiplicar por 32. 

Pero para entender un poco mas hay que descomponer 33. El numero 33 se puede escribir como ```32 + 1```, simplemente por que una propiedad basica de la aritmetica es que cualquier numero puede desomponerse en la suma de dos numeros, por lo tanto multiplicar cualquier numero por 33 es lo mismo que multiplicar por 32 y luego sumarle el mismo numero una vez. Y eso exactamente lo mismo que ```(hash  << 5) + hash``` desplazar un numero *N* posiciones a la izquierda es igual a multiplicar ese numero por *2^n*, y para acompletar a 33 en djb2 se suma el valor original del hash, lo cual lo hace mas eficiente, haciendo que los numeros 33 y 5381 sean practicamente perfectos.

Aun que esta es una explicacion "razonable", en realidad aun no se explica adecuadamente la razon de esos numeros.

## Implementacion

```c
#include <stdio.h>

unsigned long djb2(unsigned char *str) {
    unsigned long hash = 5381;
    int c;
    
    while ((c = *str++)) {
        hash = ((hash << 5) + hash) + c; /* hash * 33 + c */
    }
    
    return hash;
}


int main() {
    unsigned char texto[] = "Nazuna";
    unsigned long hash = djb2(texto);

    printf("Hash de '%s' -> %lu\n", texto, hash);

    return 0;
}
```

El proceso de hasheado es ```hash = ((hash << 5) + hash) + c;```, simplemente al resultado de desplazar 5 bits a la izquierda del hash, se le suma el valor del hash original , por ultimo se suma el valor en ASCII del caracter actual. Este bucle se recorre hasta que que *str apunte a ```'\0'```

```bash
❯❯❯ ./djb2
Hash de 'Nazuna' -> 6952561251634
```
