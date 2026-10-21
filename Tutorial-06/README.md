# randomquotes (Tutorial 06)

Proyecto Express que devuelve una frase aleatoria en `/`.

## Correr localmente

```
npm install
node index.js
```

La app escucha en el puerto 80: http://localhost/

## Docker (VM en GCP)

```
git clone https://github.com/davidosoriot/davidosoriot-topicos-software.git
cd davidosoriot-topicos-software/Tutorial-06
docker build -t randomquotes .
docker run -d -p 80:80 --name randomquotes randomquotes
```

Luego abrir `http://<IP-de-la-VM>/`.
