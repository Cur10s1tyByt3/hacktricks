# Rust Basics

{{#include ../banners/hacktricks-training.md}}

### Types génériques

Créez une struct où 1 de leurs valeurs pourrait être de n'importe quel type
```rust
struct Wrapper<T> {
value: T,
}

impl<T> Wrapper<T> {
pub fn new(value: T) -> Self {
Wrapper { value }
}
}

Wrapper::new(42).value
Wrapper::new("Foo").value, "Foo"
```
### Option, Some & None

Le type Option signifie que la valeur peut être de type Some (il y a quelque chose) ou None :
```rust
pub enum Option<T> {
None,
Some(T),
}
```
Vous pouvez utiliser des fonctions telles que `is_some()` ou `is_none()` pour vérifier la valeur de l'Option.

### Macros

Les macros sont plus puissantes que les fonctions car elles s'étendent pour produire plus de code que le code que vous avez écrit manuellement. Par exemple, une signature de fonction doit déclarer le nombre et le type de paramètres que la fonction a. Les macros, en revanche, peuvent prendre un nombre variable de paramètres : nous pouvons appeler `println!("hello")` avec un argument ou `println!("hello {}", name)` avec deux arguments. De plus, les macros sont étendues avant que le compilateur n'interprète le sens du code, donc une macro peut, par exemple, implémenter un trait sur un type donné. Une fonction ne peut pas le faire, car elle est appelée à l'exécution et un trait doit être implémenté à la compilation.
```rust
macro_rules! my_macro {
() => {
println!("Check out my macro!");
};
($val:expr) => {
println!("Look at this other macro: {}", $val);
}
}
fn main() {
my_macro!();
my_macro!(7777);
}

// Export a macro from a module
mod macros {
#[macro_export]
macro_rules! my_macro {
() => {
println!("Check out my macro!");
};
}
}
```
### Itérer
```rust
// Iterate through a vector
let my_fav_fruits = vec!["banana", "raspberry"];
let mut my_iterable_fav_fruits = my_fav_fruits.iter();
assert_eq!(my_iterable_fav_fruits.next(), Some(&"banana"));
assert_eq!(my_iterable_fav_fruits.next(), Some(&"raspberry"));
assert_eq!(my_iterable_fav_fruits.next(), None); // When it's over, it's none

// One line iteration with action
my_fav_fruits.iter().map(|x| capitalize_first(x)).collect()

// Hashmap iteration
for (key, hashvalue) in &*map {
for key in map.keys() {
for value in map.values() {
```
### Boîte Récursive
```rust
enum List {
Cons(i32, List),
Nil,
}

let list = Cons(1, Cons(2, Cons(3, Nil)));
```
### Conditionnels

#### si
```rust
let n = 5;
if n < 0 {
print!("{} is negative", n);
} else if n > 0 {
print!("{} is positive", n);
} else {
print!("{} is zero", n);
}
```
#### correspondre
```rust
match number {
// Match a single value
1 => println!("One!"),
// Match several values
2 | 3 | 5 | 7 | 11 => println!("This is a prime"),
// TODO ^ Try adding 13 to the list of prime values
// Match an inclusive range
13..=19 => println!("A teen"),
// Handle the rest of cases
_ => println!("Ain't special"),
}

let boolean = true;
// Match is an expression too
let binary = match boolean {
// The arms of a match must cover all the possible values
false => 0,
true => 1,
// TODO ^ Try commenting out one of these arms
};
```
#### boucle (infinie)
```rust
loop {
count += 1;
if count == 3 {
println!("three");
continue;
}
println!("{}", count);
if count == 5 {
println!("OK, that's enough");
break;
}
}
```
#### pendant
```rust
let mut n = 1;
while n < 101 {
if n % 15 == 0 {
println!("fizzbuzz");
} else if n % 5 == 0 {
println!("buzz");
} else {
println!("{}", n);
}
n += 1;
}
```
#### pour
```rust
for n in 1..101 {
if n % 15 == 0 {
println!("fizzbuzz");
} else {
println!("{}", n);
}
}

// Use "..=" to make inclusive both ends
for n in 1..=100 {
if n % 15 == 0 {
println!("fizzbuzz");
} else if n % 3 == 0 {
println!("fizz");
} else if n % 5 == 0 {
println!("buzz");
} else {
println!("{}", n);
}
}

// ITERATIONS

let names = vec!["Bob", "Frank", "Ferris"];
//iter - Doesn't consume the collection
for name in names.iter() {
match name {
&"Ferris" => println!("There is a rustacean among us!"),
_ => println!("Hello {}", name),
}
}
//into_iter - COnsumes the collection
for name in names.into_iter() {
match name {
"Ferris" => println!("There is a rustacean among us!"),
_ => println!("Hello {}", name),
}
}
//iter_mut - This mutably borrows each element of the collection
for name in names.iter_mut() {
*name = match name {
&mut "Ferris" => "There is a rustacean among us!",
_ => "Hello",
}
}
```
#### si laissez
```rust
let optional_word = Some(String::from("rustlings"));
if let word = optional_word {
println!("The word is: {}", word);
} else {
println!("The optional word doesn't contain anything");
}
```
#### while let
```rust
let mut optional = Some(0);
// This reads: "while `let` destructures `optional` into
// `Some(i)`, evaluate the block (`{}`). Else `break`.
while let Some(i) = optional {
if i > 9 {
println!("Greater than 9, quit!");
optional = None;
} else {
println!("`i` is `{:?}`. Try again.", i);
optional = Some(i + 1);
}
// ^ Less rightward drift and doesn't require
// explicitly handling the failing case.
}
```
### Traits

Créez une nouvelle méthode pour un type
```rust
trait AppendBar {
fn append_bar(self) -> Self;
}

impl AppendBar for String {
fn append_bar(self) -> Self{
format!("{}Bar", self)
}
}

let s = String::from("Foo");
let s = s.append_bar();
println!("s: {}", s);
```
### Tests
```rust
#[cfg(test)]
mod tests {
#[test]
fn you_can_assert() {
assert!(true);
assert_eq!(true, true);
assert_ne!(true, false);
}
}
```
### Threading

#### Arc

Un Arc peut utiliser Clone pour créer plus de références sur l'objet afin de les passer aux threads. Lorsque le dernier pointeur de référence vers une valeur sort de la portée, la variable est supprimée.
```rust
use std::sync::Arc;
let apple = Arc::new("the same apple");
for _ in 0..10 {
let apple = Arc::clone(&apple);
thread::spawn(move || {
println!("{:?}", apple);
});
}
```
#### Threads

Dans ce cas, nous passerons au thread une variable qu'il pourra modifier.
```rust
fn main() {
let status = Arc::new(Mutex::new(JobStatus { jobs_completed: 0 }));
let status_shared = Arc::clone(&status);
thread::spawn(move || {
for _ in 0..10 {
thread::sleep(Duration::from_millis(250));
let mut status = status_shared.lock().unwrap();
status.jobs_completed += 1;
}
});
while status.lock().unwrap().jobs_completed < 10 {
println!("waiting... ");
thread::sleep(Duration::from_millis(500));
}
}
```
### Essentials de sécurité

Rust fournit de fortes garanties de sécurité mémoire par défaut, mais vous pouvez toujours introduire des vulnérabilités critiques par le biais de code `unsafe`, de problèmes de dépendance ou d'erreurs logiques. Le mini-cheat sheet suivant rassemble les primitives que vous rencontrerez le plus souvent lors des revues de sécurité offensives ou défensives des logiciels Rust.

#### Code unsafe & sécurité mémoire

Les blocs `unsafe` se soustraient aux vérifications d'aliasing et de limites du compilateur, donc **tous les bugs traditionnels de corruption de mémoire (OOB, use-after-free, double free, etc.) peuvent réapparaître**. Une liste de contrôle rapide pour l'audit :

* Recherchez des blocs `unsafe`, des fonctions `extern "C"`, des appels à `ptr::copy*`, `std::mem::transmute`, `MaybeUninit`, des pointeurs bruts ou des modules `ffi`.
* Validez chaque opération arithmétique sur les pointeurs et chaque argument de longueur passé aux fonctions de bas niveau.
* Préférez `#![forbid(unsafe_code)]` (à l'échelle du crate) ou `#[deny(unsafe_op_in_unsafe_fn)]` (1.68 +) pour échouer la compilation lorsque quelqu'un réintroduit `unsafe`.

Exemple de dépassement créé avec des pointeurs bruts :
```rust
use std::ptr;

fn vuln_copy(src: &[u8]) -> Vec<u8> {
let mut dst = Vec::with_capacity(4);
unsafe {
// ❌ copies *src.len()* bytes, the destination only reserves 4.
ptr::copy_nonoverlapping(src.as_ptr(), dst.as_mut_ptr(), src.len());
dst.set_len(src.len());
}
dst
}
```
Exécuter Miri est un moyen peu coûteux de détecter les UB au moment des tests :
```bash
rustup component add miri
cargo miri test  # hunts for OOB / UAF during unit tests
```
#### Auditing dependencies with RustSec / cargo-audit

La plupart des vulnérabilités Rust dans le monde réel se trouvent dans des crates tierces. La base de données des avis RustSec (alimentée par la communauté) peut être interrogée localement :
```bash
cargo install cargo-audit
cargo audit              # flags vulnerable versions listed in Cargo.lock
```
Intégrez-le dans CI et échouez sur `--deny warnings`.

`cargo deny check advisories` offre une fonctionnalité similaire ainsi que des vérifications de licence et de liste d'interdiction.

#### Vérification de la chaîne d'approvisionnement avec cargo-vet (2024)

`cargo vet` enregistre un hachage de révision pour chaque crate que vous importez et empêche les mises à jour non remarquées :
```bash
cargo install cargo-vet
cargo vet init      # generates vet.toml
cargo vet --locked  # verifies packages referenced in Cargo.lock
```
L'outil est adopté par l'infrastructure du projet Rust et un nombre croissant d'organisations pour atténuer les attaques par paquets empoisonnés.

#### Fuzzing votre surface API (cargo-fuzz)

Les tests de fuzzing détectent facilement les panics, les débordements d'entiers et les bogues logiques qui pourraient devenir des problèmes de DoS ou de canaux auxiliaires :
```bash
cargo install cargo-fuzz
cargo fuzz init              # creates fuzz_targets/
cargo fuzz run fuzz_target_1 # builds with libFuzzer & runs continuously
```
Ajoutez la cible de fuzz à votre dépôt et exécutez-la dans votre pipeline.

## Références

- RustSec Advisory Database – <https://rustsec.org>
- Cargo-vet: "Auditing your Rust Dependencies" – <https://mozilla.github.io/cargo-vet/>

{{#include ../banners/hacktricks-training.md}}
