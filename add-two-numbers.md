# Suma de dos números representados como listas enlazadas (orden inverso)

## Resumen ejecutivo

Dadas dos listas enlazadas no vacías `l1` y `l2` que representan números no negativos, donde **cada nodo contiene un dígito** y los dígitos están almacenados en **orden inverso** (la cabeza es la unidad), se requiere **sumar** ambos números y **devolver el resultado** como una nueva lista enlazada también en orden inverso.

---

## Modelo de datos

Cada nodo de la lista enlazada tiene la forma:

```java
class ListNode {
    int val;        // dígito en [0..9]
    ListNode next;  // referencia al siguiente nodo
}
```

Las listas **no** tienen ceros a la izquierda (excepto el número 0 en sí) y su longitud está en `[1, 100]`.

---

## Idea clave (invariante)

Recorremos las dos listas **simultáneamente** manteniendo un **acarreo** (`carry`) entre 0 y 1 (o, en general, 0..9 si hubiese más listas).
En cada iteración, la suma parcial es:

```
sum = (valor de l1 si existe, si no 0) + (valor de l2 si existe, si no 0) + carry
```

El nuevo dígito del resultado es `sum % 10` y el nuevo `carry` es `sum / 10` (división entera).
Se crea un nodo con ese dígito y se enlaza al final del resultado. El proceso continúa hasta que **ambas** listas se agotan **y** `carry == 0`.

**Invariante:** después de procesar k nodos (k ≥ 0), la lista resultado almacena exactamente los **k dígitos menos significativos** de la suma de los prefijos de `l1` y `l2`, y `carry` representa el acarreo pendiente hacia el siguiente dígito.

---

## Algoritmo paso a paso

1. **Inicialización**

   * Crear un **nodo centinela** `dummyHead` con valor 0 para simplificar el manejo de punteros.
   * Definir un puntero `current = dummyHead` que apunta al último nodo del resultado.
   * Definir `carry = 0`.
   * Definir punteros de lectura `p = l1` y `q = l2`.

2. **Bucle principal**

   * Mientras `p != null` **o** `q != null` **o** `carry != 0`:

     1. Leer `x = (p != null) ? p.val : 0`.
     2. Leer `y = (q != null) ? q.val : 0`.
     3. Calcular `sum = x + y + carry`.
     4. Actualizar `carry = sum / 10` (entero).
     5. Determinar `digit = sum % 10`.
     6. Crear un nuevo nodo `node = new ListNode(digit)`.
     7. Enlazar: `current.next = node` y avanzar `current = node`.
     8. Avanzar `p = p.next` si `p != null`; avanzar `q = q.next` si `q != null`.

3. **Finalización**

   * Devolver `dummyHead.next` (omitimos el centinela).

---

## Diagrama (flujo por iteración)

```
p.val ─┐
       ├──► sum = x + y + carry ──► digit = sum % 10 ──► [append digit]
q.val ─┘                            carry = sum / 10 ──► (siguiente iteración)
```

---

## Ejemplo detallado

### Entrada

* `l1 = [2,4,3]`  ⇒ número 342
* `l2 = [5,6,4]`  ⇒ número 465

### Iteraciones

1. `x=2, y=5, carry=0` → `sum=7` → `digit=7`, `carry=0` → resultado: `[7]`
2. `x=4, y=6, carry=0` → `sum=10` → `digit=0`, `carry=1` → resultado: `[7,0]`
3. `x=3, y=4, carry=1` → `sum=8` → `digit=8`, `carry=0` → resultado: `[7,0,8]`
4. Listas agotadas y `carry=0` → fin.

### Salida

`[7,0,8]` (que representa 807).

---

## Complejidad

* **Tiempo:** `O(max(m, n))`, donde `m = len(l1)` y `n = len(l2)`. Se visita cada nodo a lo sumo una vez.
* **Espacio adicional:** `O(1)` (sin contar la lista resultado), ya que se usan unas pocas variables y se crea exactamente un nodo por dígito del resultado.

---

## Demostración informal de corrección

* **Totalidad:** El bucle continúa mientras haya dígitos por procesar o un acarreo pendiente.
* **Preservación del invariante:** En cada paso, agregamos al resultado el dígito menos significativo de la suma parcial y propagamos el resto como `carry`.
* **Finalización:** Como las listas son finitas y `carry` se reduce como máximo a 1 por iteración, eventualmente `p == q == null` y `carry == 0`, terminando el bucle.

---

## Implementación en Java (idiomática)

```java
public class ListNode {
    int val;
    ListNode next;
    ListNode() {}
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode dummyHead = new ListNode(0);
        ListNode current = dummyHead;
        int carry = 0;

        while (l1 != null || l2 != null || carry != 0) {
            int x = (l1 != null) ? l1.val : 0;
            int y = (l2 != null) ? l2.val : 0;

            int sum = x + y + carry;
            carry = sum / 10;
            int digit = sum % 10;

            current.next = new ListNode(digit);
            current = current.next;

            if (l1 != null) l1 = l1.next;
            if (l2 != null) l2 = l2.next;
        }
        return dummyHead.next;
    }
}
```

---

## Casos borde a considerar

1. **Listas de distinta longitud:** p. ej., `l1=[9,9,9]`, `l2=[1]` → se siguen sumando con `x=0` o `y=0` al agotarse una lista.
2. **Acarreo al final:** p. ej., `l1=[5]`, `l2=[5]` → resultado `[0,1]` (10).
3. **Una lista es cero:** `l1=[0]`, `l2=[7,3]` → resultado `[7,3]`.
4. **Todas cifras 9:** `l1=[9,9,9,9,9,9,9]`, `l2=[9,9,9,9]` → `[8,9,9,9,0,0,0,1]`.
5. **Entrada mínima:** `l1=[0]`, `l2=[0]` → `[0]`.

---

## Pruebas sugeridas (tablas)

| l1                | l2        | salida esperada   |
| ----------------- | --------- | ----------------- |
| [2,4,3]           | [5,6,4]   | [7,0,8]           |
| [0]               | [0]       | [0]               |
| [9,9,9,9,9,9,9]   | [9,9,9,9] | [8,9,9,9,0,0,0,1] |
| [1,8]             | [0]       | [1,8]             |
| [5]               | [5]       | [0,1]             |
| [1,0,0,0,0,0,0,1] | [5,6,4]   | [6,6,4,0,0,0,0,1] |

---

## Errores comunes (y cómo evitarlos)

* **Olvidar el `carry`** al final del bucle → use la condición `while (l1 != null || l2 != null || carry != 0)`.
* **No enlazar correctamente** el nuevo nodo al resultado → siempre `current.next = new ListNode(digit); current = current.next;`.
* **Usar estructuras auxiliares innecesarias** (como `ArrayList`) → construya la lista sobre la marcha con un nodo centinela.
* **Confundir el orden** de los dígitos → recuerde que el nodo cabeza es la **unidad** (orden inverso).

---

## Variantes y extensiones

* **Listas en orden directo (MSD primero):** requiere invertir listas, o usar pila/recursión para sumar desde el final.
* **Suma de *k* listas:** iterar acumulando `sum` con todos los dígitos disponibles y `carry` generalizado.
* **Arreglos en lugar de listas:** misma lógica, pero con índices y sin nodos.

---

## Complejidad espacial del resultado

El tamaño de la lista resultado es **a lo sumo `max(m, n) + 1`** (por posible acarreo final). Este espacio es **necesario** para representar el número suma.

---

## Pseudocódigo (neutral al lenguaje)

```
function addTwoNumbers(l1, l2):
    dummy ← new Node(0)
    curr  ← dummy
    carry ← 0
    p ← l1
    q ← l2
    while p ≠ null or q ≠ null or carry ≠ 0:
        x ← (p ≠ null) ? p.val : 0
        y ← (q ≠ null) ? q.val : 0
        s ← x + y + carry
        carry ← s div 10
        digit ← s mod 10
        curr.next ← new Node(digit)
        curr ← curr.next
        if p ≠ null: p ← p.next
        if q ≠ null: q ← q.next
    return dummy.next
```

---

## Consideraciones de estilo/limpieza (Java)

* Usar `dummyHead` evita lógica condicional para el primer nodo.
* Mantener variables con nombres cortos pero claros (`x`, `y`, `sum`, `carry`).
* Limitar el ámbito de variables al bucle cuando sea posible.
* Evitar cajas (`Integer`) y preferir primitivos (`int`).

---

## Conclusión

El algoritmo es lineal, in-situ en términos de memoria adicional, y elegante gracias al uso de un **nodo centinela** y un **acarreo** explícito. Maneja correctamente longitudes desiguales y acarreo final, cumpliendo las restricciones del problema.

