# Codando um servidor web (extremamente simples) em Rust

Eu estava querendo aprender Rust a um tempo, e após dar uma lida [docs do Rust](https://doc.rust-lang.org/book/) e entender a sintaxe básica, decidi embarcar em um pequeno projeto baseado [neste](http://joelmccracken.github.io/entries/a-simple-web-app-in-rust-pt-1/).

## O app (mudar titulo)

O app terá uma funcionalidade muito simples: _servir_ arquivos .html ao navegador quando rodando. Será parecido com a extenção [Live server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) do vscode (só que um tanto mais burro).

O app usará a _crate_ [Nickel](https://nickel-org.github.io/) do Rust. Esta framework serve para criar aplicativos web com o Rust.

> O Nickel seria similar com algo como Django no Python ou Express no JavaScript

## O principio

No site do Nickel há um `Hello world!` usando a framework, vamos ver no que dá.

```rust
#[macro_use] extern crate nickel;

use nickel::Nickel;

fn main() {
    let mut server = Nickel::new();

    server.utilize(router! {
        get "**" => |_req, _res| {
            "Hello world!"
        }
    });

    server.listen("127.0.0.1:6767");
}
```

Ok, pelo visto dentro da função main o server é instanciado com o `Nickel::new()`.
Depois parece que o servidor "usa" (`.utilize()`) o roteador (`router!{...}`) com as rotas definidas entre as chaves... hmm, ou é isso que imagino estar acontecendo.

Vamos rodar.

```shell
cargo run
...
Listening on http://127.0.0.1:6767
Ctrl-C to shutdown server
```

Deu certo!! Se eu for em `localhost:6767` no navegador, `Hello world!` está escrito no topo da página.

Mas, quando eu executei, ocorreu um aviso:

```shell
warning: unused `Result` that must be used
  --> src\main.rs:14:5
   |
14 |     server.listen("127.0.0.1:6767");
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: this `Result` may be an `Err` variant, which should be handled
   = note: `#[warn(unused_must_use)]` on by default
help: use `let _ = ...` to ignore the resulting value
   |
14 |     let _ = server.listen("127.0.0.1:6767");
   |     +++++++

warning: `web_server` (bin "web_server") generated 1 warning
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.36s
```

Hmm, lembro de ter visto algo assim nos docs do Rust, aparentemente `server.listen()` retorna um _Result_, e deve ser tratado caso tenha um erro.
Lembro de ter usado o `.expect()` quando estava aprendendo, vamos ver se da certo aqui:

```rust
...
fn main() {
    ...
    server.listen("127.0.0.1:6767").expect("Um erro ocorreu ao iniciar o servidor.");
}
```

Aí sim, sem _warnings_! Se um erro ocorrer, o programa vai parar com a mensagem "Um erro ocorreu ao iniciar o servidor.", Perfeito!

## Retornando Html

Legal, vamos testar se retornar html funciona direto, ou vamos ter que fazer alguma tramoia:

```rust
...
fn main() {
    let mut server = Nickel::new();

    server.utilize(router! {
        get "**" => |_req, _res| {
            "<h1>Teste teste</h1>"
        }
    });

    server.listen("127.0.0.1:6767").expect("Um erro ocorreu ao iniciar o servidor.");
}
```

Retorna sim! o texto dentro do h1 virou um cabeçalho, ótimo, menos trabalho para mim.

Próximo passo, ler html de um arquivo .html.

## Lendo html de um arquivo .html

Primeiramente vamos criar um arquivo .html:

```html
index.html

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <h1>Meu arquivo html!</h1>
    <p>Teste Teste</p>
  </body>
</html>
```

Ok, a minha ideia é a seguinte: No Python tem a função `open()` que abre possibilita a leitura deles. Eu imagino que no Rust deve haver algo parecido...

E após uma pequena pesquisa, parece que tem sim, é só fazer uso da biblioteca `fs`, vamos lá:

```rust
#[macro_use] extern crate nickel;

use std::fs;
use nickel::Nickel;

fn main() {
    let mut server = Nickel::new();

    server.utilize(router! {
        get "**" => |_req, _res| {
            fs::read_to_string("index.html")
        }
    });

    server.listen("127.0.0.1:6767").expect("Um erro ocorreu ao iniciar o servidor.");
}
```

Depoisq ue escrevi esse código meu vscode acendeu que nem uma árvore de natal. Pelo jeito, a função `read_to_string()` retorna um _Result_, parecido com o warning anteriror do `server.listen()`, então vou consertar do mesmo jeito:

```rust
fs::read_to_string("index.html").expect("Erro ao ler o arquivo.")
```

E vamo que vamo, deu certo! O arquivo html é mostrado na tela normalmente.

Ta, mas ta meio feio isso, vamos ver se conseigo colocar essa leitura numa função:

```rust
#[macro_use] extern crate nickel;

use std::fs;
use nickel::Nickel;

//  mismatched types
//  expected unit type `()`
//          found enum `Result<String, std::io::Error>
fn read_file(file_path: &str) {
    return fs::read_to_string(file_path);
}

fn main() {
    let mut server = Nickel::new();

    server.utilize(router! {
        get "**" => |_req, _res| {
            read_file("index.html").expect("Erro ao ler o arquivo.")
        }
    });

    server.listen("127.0.0.1:6767").expect("Um erro ocorreu ao iniciar o servidor.");
}
```

Huh, quando escrevo esta função, da um problema no tipo ... do retorno? Deve ser por que antes eu estava usando o `.expect()` para concertar esse problema. Ao passar meu mouse por cima do erro, o vscode me da uma dica (deve ser da extenção de Rust que eu baixei):

```
main.rs(6, 30): try adding a return type: ` -> Result<String, std::io::Error>`
```

Ah, então é isso mesmo, o problema está no retorno, adicionar o tipo do retorno na função fica algo como:

```rust
...
fn read_file(file_path: &str) -> Result<String, std::io::Error>{
    return fs::read_to_string(file_path);
}
...
```

E rodando esse código, tudo funciona como o esperado!

## Deixando dinâmico

Ok, proximo passo: temos que fazer o programa retornar o arquivo html solicitado pelo navegador.
Temos que analisar a [URI](https://woliveiras.com.br/posts/url-uri-qual-diferenca) que para o nosso exemplo é simplesmente tudo que vem depois da URL.

Se o link no navegador for `localhost:6767/meuhtml.html`, o servidor deve ler e mostrar na tela o conteúdo do arquivo `meuhtml.html`, parece simples.

Após uma pesquisada, descobri que posso a variável `_req`, para pegar os valores digitados, algo como :

```rust
...
server.utilize(router! {
        get "**" => |_req, _res| {
            let file_path = _req.origin.uri.to_string();
            println!("{:?}", file_path);
            // expected &str, found String
            read_file(file_path).expect("Erro ao ler o arquivo.")
        }
    });
...
```

Um erro ocorre, `expected &str, found String`, pelo visto `_req.origin.uri.to_string()` retorna uma String não uma &str, vamos mudar o parâmetro da função `read_file()` para `String` também então.

```rust
...
fn read_file(file_path: String) -> Result<String, std::io::Error>{
    return fs::read_to_string(file_path);
}
...
```

E com isso temos um novo erro! Progresso! O `.expect()` é executado e nos diz o seguinte:

```shell
Listening on http://127.0.0.1:6767
Ctrl-C to shutdown server
"/"
thread '<unnamed>' panicked at src\main.rs:18:34:
Erro ao ler o arquivo.: Os { code: 3, kind: NotFound, message: "O sistema não pode encontrar o caminho especificado." }
```

Inspecionando esse resultado, o erro é óbvio: Note o "/" na terceira linha, este é o resultado do `println!("{:?}", file_path);`. E é claro que o arquivo "/" não existe!

Isso denota uma conveção da web: Quando a rota termina em "/" devemos procurar o arquivo `index.html`. Por exemplo, `localhost:6767/meusite/` deve retornar o arquivo `meusite/index.html`.
