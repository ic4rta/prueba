---
layout: default
title: Excepciones
parent: Perl
grand_parent: Programacion
---

# Excepciones
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## eval

Para manejar excepciones en perl de forma nativa podemos hace uso de ```eval``` y ```$@```, la sintaxis general es asi:

```perl
local $@;
eval {
	codigo();
};
if ($@){
	warn "Error --> [$@]";
	$@ = "";
	return undef;
}
```
eval se usa para envolver y ejecutar el codigo que puede ocasionar una excepcion, si el codigo falla, el error se guarda en ```$@```, en el warn se le indica la variable global  ```$@``` para que imprima la excepcion de eval (si es que ocasiono una).

Aqui podemos ver un ejemplo para manejar excepciones sobre la apertura de un archivo

```perl
sub abrir_archivo {
	my ($archivo) = @_;
	
	my $contenido = "";
	local $@;
	eval {
		open(ARCHIVO, "<", $archivo) or die "No se puede abrir el $archivo";
		while(<ARCHIVO>){
			$contenido .= $_;
		} 
		close(ARCHIVO);
	};
	if ($@){
		warn "I/O Excepcion --> [$@]";
		$@ = "";
		return undef;
	}
	return $contenido;
}

sub main {
	print abrir_archivo("/tmp/archivo-2.tx");
}
main()
```

En este caso se ejecutaran dos exepciones, el ```die``` de la funcion y el ```warn```, y parece un poco redundante ejectuar dos excepciones e incluso es mas codigo, y si, pero la idea de hacerlo asi es conseguir mas robustes en el codigo 
En este caso cuando se llame a ```warn``` tambien imprimira el mensaje de contenga ```$@```, como recomendacion, en el ```warn``` tengan una excepcion mas general y en los ```die``` una mas especifica

Por ejemplo, si trabajan con archivos, en el warn pueden ingresar algo como ```I/O Error``` o ```File Exception```, mientras que en el die pueden ingresar una excepcion mas especifica como puede ser ```El archivo <archivo> no existe```

Aqui esta otro ejemplo para capturar la excepcion de que no se puede dividir entre 0

```perl
sub dividir {
	my ($numero_1, $numero_2) = @_;
	my $resultado;
	
	local $@;
	eval {
		if ($numero_2 == 0){
			die "No se puede dividir entre 0";
		}else{
			$resultado = $numero_1 / $numero_2;
		}
	};
	if ($@){
		warn "Math Excepcion: [$@]";
		$@ = "";
		return undef;
	}
	return $resultado;
}


sub main {
	print dividir(1, 0);
}

main()
```
Lo que regresaria el codigo al obtener la excepcion es:
```
Math Excepcion --> [No se puede dividir entre 0 at error-math.pl line 8.
] at error-math.pl line 14.
```

## Exception::Class
Es un modulo que permite crear excepciones de una manera mas robusta y segura, ademas tiene la ventaja de que nosotros creamos nuestras propias excepciones para cualquier caso. La sintaxis basica puede ser

```perl
use Exception::Class (
	"Error::NombreExcepcion" => {
		description => "La razon de la excepcion",
		fields => ["atributo adicional"]
	}
);
```

Como ejemplo, se tomara el que se hizo anteriormente del archivo pero agregando que si no tenemos permisos de lectura:

```perl
use Exception::Class (
	"Error::ArchivoNoEncontrado" => {
		description => "El archivo no existe",
		fields => ["nombre"]
	},
    "Error::PermisoDenegado" => {
        description => "Permiso denegado para abrir el archivo",
        fields => ["nombre"]
    }	
);

sub abrir_archivo {
	my ($archivo) = @_;
	my $contenido;
	
	open(ARCHIVO, "<", $archivo) or do {
		if (!-e $archivo) {
			Error::ArchivoNoEncontrado->throw(
				error => "El $archivo no existe",
				nombre => $archivo,
			);
		} elsif (!-r $archivo) {
	        Error::PermisoDenegado->throw(
	            error => "No tienes permisos de lectura para $archivo",
	            nombre => $archivo,
	        );
		}
	};
	
	while(<ARCHIVO>){
		$contenido .= $_;
	}
	
	close(ARCHIVO);
	return $contenido;
}


sub main {
	eval {
		print abrir_archivo("/tmp/archivo-2.txt");
	};

	if (my $e = Error::ArchivoNoEncontrado->caught) {
		warn "Error::ArchivoNoEncontrado --> " . $e->error;
	} elsif (my $e = Error::PermisoDenegado->caught) {
		warn "Error:PermisoDenegado --> " . $e->error;
	}
}

main()
```

Ahora la salida de cuando ocurran una de estas excepciones era algo asi:
```
Error:PermisoDenegado --> No tienes permisos de lectura para /tmp/archivo-2.txt at custom-error.pl line 47.
```

En este caso se usa ```eval```en el main, pero la propia documentación nos recomienda usar ```Try::Tiny```

```perl
use Exception::Class (
	"Error::ArchivoNoEncontrado" => {
		description => "El archivo no existe",
		fields => ["nombre"]
	},
    "Error::PermisoDenegado" => {
        description => "Permiso denegado para abrir el archivo",
        fields => ["nombre"]
    }	
);
use Try::Tiny;

sub abrir_archivo {
	my ($archivo) = @_;
	my $contenido;
	
	open(ARCHIVO, "<", $archivo) or do {
		if (!-e $archivo) {
			Error::ArchivoNoEncontrado->throw(
				error => "El $archivo no existe",
				nombre => $archivo,
			);
		} elsif (!-r $archivo) {
	        Error::PermisoDenegado->throw(
	            error => "No tienes permisos de lectura para $archivo",
	            nombre => $archivo,
	        );
		}
	};
	
	while(<ARCHIVO>){
		$contenido .= $_;
	}
	
	close(ARCHIVO);
	return $contenido;
}


sub main {
    try {
        print abrir_archivo("/tmp/archivo-2.txt");
    } catch {
        if (blessed $_ && $_->can("isa")) {
            if ($_->isa('Error::ArchivoNoEncontrado')) {
                warn "Error::ArchivoNoEncontrado --> " . $_->error;
            }
            elsif ($_->isa('Error::PermisoDenegado')) {
                warn "Error::PermisoDenegado --> " . $_->error;
            }
        }
    };
}

main()
```
En este caso es necesario usar ```if (blessed $_ && $_->can('isa'))``` para verificar si ```$_``` es una instancia de una excepción definida con ```Exception::Class```

Se usar ```$_->can("isa")``` para verificar si ```$_``` tiene tiene el método ```isa```, normalmente isa se usa para verificar si un objeto es instancia de una clase especifica

Pos otra parte se usa ```blessed``` o bendecir, donde basicamente se dice que esta "bendecido" si una variable es una referencia a un objeto. Este metodo devuelve el nombre de la clase a la que pertenece al objeto "bendecido"

## Throwable y Try::Tiny

Personalmente esta es una de las maneras que que mas me gusta crear excepciones

Para hacerlo se definen dos archivos, uno con extension .pm y el .pl, en el .pm debemos de ingresar nuestras excepciones, como por ejemplo:

```perl
package FileNotFound;
use base 'Throwable::Error';

sub error {
    my $self = shift;
    return "Archivo inexistente: " . $self->message;
}

package FileAccess;
use base 'Throwable::Error';

sub error {
    my $self = shift;
    return "Permisos insuficientes: " . $self->message;
}

1;
```
En package debemos de indicar el nombre de nuestra excepción, no importa si es español o ingles, etc, posteriormente definimos el método ```error``` el en el cual yo siempre suelo definir un mensaje mas generalizado, pero es a decisión personal. Algo tambien importante es que se le debe de incluir el atributo ```message``` para llamar a la excepcion

El archivo principal que llamaria a las excepciones quedaria asi:

```perl
use Try::Tiny;
use FindBin;
use lib "$FindBin::Bin";
use Excepciones;
use Scalar::Util 'blessed';

sub abrir_archivo {
    my $archivo = shift;

    try {
        open(ARCHIVO, '<', $archivo)
            or do {
                if (!-e $archivo) {
                    FileNotFound->throw(
                    	message => "El archivo '$archivo' no existe"
                    );
                } elsif (!-r $archivo) {
                    FileAccess->throw(
                    	message => "No tienes permisos de lectura sobre '$archivo'"
                    );
                }
            };

        while (<ARCHIVO>) {
            print $_;
        }

        close(ARCHIVO);
    } catch {
        if (blessed($_) && $_->isa('Throwable::Error')) {
            warn "Excepción I/O --> " . $_->error . "\n";
        }
    };
}

sub main{
	abrir_archivo("prueba.txt");
	abrir_archivo("/etc/shadow");	
}

main()
```

Es importante que incluir nuestro modulo de las excepciones ```use Excepciones```, ese modulo lo debemos de incluir respetando el nombre del archivo .pm.

Y para llamar a la excepcion usamos el nombre de la excepcion junto con la palabra throw

```perl
NombreExcepcion->throw(
	message => "Mensaje mas especifico de la excepcion"
);
```

En este ejemplo cuando ocurra una excepción se vera asi:

```
Excepción I/O --> Permisos insuficientes: No tienes permisos de lectura sobre '/etc/shadow'
```