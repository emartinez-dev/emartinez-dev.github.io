---
title:  "miniRT: otro renderizador 3D en C, ahora con raytracing"
categories: ["proyectos"]
tags: ["gráficos", "3D", "42", "miniRT", "raytracing"]
date: 2024-12-08T15:48:00+02:00
ShowToc: true
TocOpen: false
draft: true
cover:
    image: "/images/miniRT.png"
    alt: "Otro renderizador 3D en C, ahora con raytracing"
    caption: "Otro renderizador 3D en C, ahora con raytracing"
    relative: true
---

**Código fuente del proyecto**: [miniRT](https://github.com/emartinez-dev/miniRT)

Después de implementar [mi primer renderizador 3D](/posts/proyectos/renderizador-wireframe-en-c) 
, y aprender cómo funciona la librería gráfica MLX42, el segundo proyecto gráfico del currículum
de 42 es miniRT. Un ray tracer hecho en C (con su respectiva Norma), humilde pero funcional.

El principal objetivo de este proyecto es demostrarnos que no hay que ser matemático para
implementar funciones matemáticas o físicas. Este proyecto no lo hice solo, tuve la suerte de
formar equipo con el gran Juan Antonio García a.k.a. [Juan-aga](https://github.com/Juan-aga/).

## Conceptos fundamentales

### El Raytracing o trazado de rayos 

El ray tracing es una técnica de renderizado en gráficos por ordenador que simula cómo la luz
interactúa con los objetos en un entorno en 3 dimensiones. Se basa en la física del comportamiento
de la luz, como la reflexión, refracción y dispersión para producir imágenes realistas.

### Proyección de rayos

El proceso comienza proyectando rayos desde un punto de vista virtual (la cámara) hacia un espacio
tridimensional.

Cada rayo que se lanza desde la cámara, representa una línea de visión desde el observador hacia
una dirección específica en el mundo, y cada píxel de la pantalla corresponde a un rayo que
atraviesa un plano de proyección. La dirección del rayo se determina según la posición del píxel
en relación con la cámara.

### Intersecciones con los objetos de la escena

Para cada rayo, el motor gráfico calcula con qué objeto y a que distancia hace la intersección.
Si no la hay, esto significa que este rayo no intercepta ningún objeto y ese píxel se renderiza con
el color del fondo.

### Iluminación

Cuando un rayo intersecta con un objeto, se evalúan los efectos de la luz en ese punto para
determinar el color del pixel, que depende de los siguientes factores:

- El color del objeto.
- El ángulo de reflexión de la luz: se lanza un nuevo rayo en la
    dirección de la luz para calcular la intensidad con la que la luz incide sobre el objeto.
- La sombra: se lanzan rayos hacia todas las fuentes de luz para comprobar si algún objeto se
    interpone entre el objeto golpeado y la luz.
- La iluminación global: La luz indirecta que afecta a todos los objetos.

## El formato .rt

El requisito principal del proyecto es renderizar un espacio 3D, con luz ambiental, luces de punto,
cámara, esferas, cilindros y planos. Cada tipo de elemento será representado de la siguiente forma
dentro del formato .rt:

- Luz ambiental: `A`
- Luz de punto: `L`
- Cámara: `C`
- Esfera: `sp`
- Cilindro: `cy`
- Plano: `pl`

### Iluminación

Para la iluminación tenemos 2 propiedades comunes, que son el brillo y el color. El brillo es un
número en el rango `[0.0, 1.0]` que indica la intensidad de la luz, y el color es una lista de colores
RGB separados con comas, cada uno con un rango entre 0 y 255. Para el color rojo, tendríamos
`[255,0,0]`.

La única diferencia entre las luces ambientales y la luz de punto, es que las luces ambientales
afectan a la escena globalmente mientras que las otras tienen una posición fija en la escena, por lo
tanto necesitamos también un sistema de coordenadas. Las coordenadas serán introducidas como una
lista de valores, uno por cada eje x,y y z en este formato: `[-40.0,50.0,0.0]`.

### Cámara

La cámara, además de coordenadas, introduce también los conceptos de vector de orientación y el FOV
o campo de visión.

El vector de orientación es un vector de longitud 1 que representa en qué dirección mira la cámara.
Por ejemplo, un vector de orientación `[0.0, 0.0, 1.0]` indica que, si establecemos el punto de
origen en el `[0,0,0]`, la cámara "mirará" hacia el eje Z+, siendo capaz de ver los objetos cuya
posición en el eje Z sea mayor que la de la cámara.

El FOV (CDV para los puristas del castellano) es el ángulo de visión en grados, en un rango de
`[0, 180]`. Cuanto mayor sea el ángulo, mayor será también el rango de visualización de la cámara,
pero los objetos aparecerán más lejanos y deformados, como un ojo de pez.

Como referencia, el ojo humano tiene un ángulo de visión de unos 200 grados, pero solo procesamos la
información en torno a unos 40 o 60 grados.

### Objetos

Todos los objetos que vamos a representar tienen coordenadas x,y,z y color, pero cada objeto tiene
algunas propiedades únicas:

- La esfera añade el diámetro de esta.
- El plano tiene vector de orientación.
- El cilindro tiene vector de orientación, diámetro y altura.

Un ejemplo de escena completa sería el siguiente:

```shell
$> cat basic_scene.rt

A 0.5 255,255,255
C 0,0,-4 0,0,1 40
L -2,4,1 0.8 255,255,255

sp -0.5,0,3 1 96,246,24 
sp 1.1,0.30,2.57 0.1 10,10,10

pl 0,0,6 0,0,1 255,255,255
pl 0,-0.8,0 0,1,0 255,255,255

cy 0,-1,3.3 0,1,0 0.2 2 133,79,0
```

A la hora de parsear estos valores, tenemos que asegurarnos que todos los valores introducidos son
válidos y que están dentro de los límites especificados. Todas las escenas tienen que tener al menos
una cámara y una fuente de iluminación.

## Creando una librería para cálculos vectoriales

Como tenemos que realizar muchas operaciones con vectores, creamos un struct para representarlos
e implementamos todas las funciones necesarias para poder realizar operaciones con ellos.

Esta pequeña librería nos facilitó bastante el trabajo posteriormente, y estas son las funciones que
implementaba.

```c
typedef struct s_v3
{
	double	x;
	double	y;
	double	z;
}	t_v3;

/* Sum v2 to v1 */
t_v3	vec3_sum(t_v3 v1, t_v3 v2);
/* Subtract v2 to v1 */
t_v3	vec3_sub(t_v3 v1, t_v3 v2);

/* Multiply two vectors */
t_v3	vec3_multv(t_v3 v1, t_v3 v2);
/* Multiply vector by a scalar */
t_v3	vec3_multk(t_v3 v, double k);

/* Dot product of two vectors */
double	vec3_dot(t_v3 v1, t_v3 v2);
/* Cross product of two vectors */
t_v3	vec3_cross(t_v3 v1, t_v3 v2);

/* Divide v1 by v2 */
t_v3	vec3_divv(t_v3 v1, t_v3 v2);
/* Divide v by an scalar */
t_v3	vec3_divk(t_v3 v, double k);

/* returns the negative of the vector */
t_v3	vec3_negative(t_v3 v);
/* returns the squared length of the vector */
double	vec3_sqlen(t_v3 v);
/* returns the length of the vector */
double	vec3_len(t_v3 v);
/* returns the unit vector of v */
t_v3	vec3_normalize(t_v3 v);
/* returns the distance between two vectors */
double	vec3_distance(t_v3 v1, t_v3 v2);

typedef struct s_m4
{
	double	m[4][4];
}	t_m4;

/* Multiply a vector by a matrix */
t_v3	vec3_mulm(t_v3 v, t_m4 mt);
```

## Renderizando la escena: cámara, luces y objetos

Antes de pasar de la escena anterior a esta imagen, tendremos que jugar a ser matemáticos.
![Escena básica](/images/miniRT-basic-scene.png)

### Preparando la cámara

La cámara es el corazón del motor de ray tracing, ya que se encarga de la orientación en el espacio
tridimensional y de transformar las coordenadas de los píxeles de la pantalla en direcciones de la
escena. La representaremos con el siguiente struct:

```c
typedef struct s_camera
{
	t_v3	p; // posición
	t_v3	norm; // vector de dirección
	int     h_fov; // fov horizontal
	double	fov; // ángulo de fov en radianes
	double	aspect_ratio; // relación de aspecto
	t_m4	rotation; // matriz de rotación
}	t_camera;
```

#### Definición de la orientación

Para determinar la orientación de la cámara, definimos un vector arbitrario llamado `look_at`.
Este vector sirve como una referencia inicial para calcular la dirección "arriba" relativa al punto
de vista de la cámara.

```c
look_at = (t_v3){0, 0, 1};
if (c->norm.z == 1 || c->norm.z == -1)
    look_at = (t_v3){-c->norm.z, 0, 0};
```

Si la cámara apunta exactamente en la dirección del eje Z positivo o negativo, el cálculo del vector
"arriba" usando productos cruzados podría dar resultados inválidos (ya que los vectores serían
paralelos). En este caso, cambiamos `look_at` a un vector en el eje X para evitar problemas.

#### Cálculo de los vectores de referencia

A partir de `look_at` y el vector normal de la cámara (`norm`), calculamos los vectores principales
del sistema de referencia local:

- Up: Producto cruzado entre `look_at` y `norm`. Esto garantiza que el vector "arriba" sea
    perpendicular al eje de visión.
- Right: Producto cruzado entre `up` y `norm`, obteniendo así un vector perpendicular tanto al eje
    de visión como al vector "arriba".

Finalmente, normalizamos todos los vectores para garantizar que sean unitarios.

```c
up = vec3_cross(look_at, c->norm);
up = vec3_normalize(up);
right = vec3_cross(up, c->norm);
right = vec3_normalize(right);
```

#### Matriz de transformación

Para convertir puntos y rayos de las coordenadas locales de la cámara al espacio global, construimos
una **matriz de rotación y traslación**. Esta matriz combina los vectores `right`, `up` y `norm`
junto con la posición de la cámara (`c->p`).

```c
c->rotation.m[0][0] = right.x;
c->rotation.m[1][0] = right.y;
c->rotation.m[2][0] = right.z;
c->rotation.m[0][1] = up.x;
c->rotation.m[1][1] = up.y;
c->rotation.m[2][1] = up.z;
c->rotation.m[0][2] = c->norm.x;
c->rotation.m[1][2] = c->norm.y;
c->rotation.m[2][2] = c->norm.z;
c->rotation.m[0][3] = c->p.x;
c->rotation.m[1][3] = c->p.y;
c->rotation.m[2][3] = c->p.z;
```

La matriz resultante permite transformar direcciones y puntos en espacio local hacia el espacio
global y viceversa.

#### Cálculo del ratio de aspecto y campo de visión

Para prevenir que la escena proyectada se distorsione, necesitamos calcular la relación de aspecto
en base al alto y ancho de la ventana del programa.

Además, es más fácil calcular la dirección de los rayos si utilizamos la tangente del ángulo
horizontal del FOV (por eso lo dividimos entre dos) para convertir las coordenadas normalizadas de
la pantalla en direcciones 3D.

```c
c->aspect_ratio = w->m_width / (double)w->m_height;
c->fov = tan(to_radians(c->h_fov) / 2.0);
```

---------- TODO: me quedo por aquí

### Generación de rayos

#### Dirección del rayo por píxel

Cada píxel en la pantalla representa un punto en un plano virtual de proyección. Calculamos la dirección del rayo en el espacio de la cámara basándonos en las coordenadas del píxel (`x`, `y`) y los parámetros de la cámara como el **campo de visión horizontal** (`h_fov`) y la relación de aspecto.

```c
w = (2 * ((x + 0.5) / win->m_width) - 1) * c->aspect_ratio * c->fov;
h = (1 - 2 * ((y + 0.5) / win->m_height)) * c->fov;
return ((t_v3){w, h, 1});
```

#### Transformación al Espacio Global**

Usamos la matriz de rotación y traslación de la cámara para transformar la dirección calculada en el espacio de la cámara al espacio global. Finalmente, normalizamos la dirección del rayo.

```c
cam_ray.origin = c->p;
cam_ray.direction = cam_direction(x, y, c, w);
cam_ray.direction = vec3_mulm(cam_ray.direction, c->rotation);
cam_ray.direction = vec3_normalize(cam_ray.direction);
```

## Resultado final

## Mejoras que podríamos haber implementado

- Añadir más rebotes a los rayos
- Mapeo de texturas
- Threading
