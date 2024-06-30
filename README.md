## Dependencies
- [https://github.com/cagix/pandoc-lecture](pandoc-lecture)

## How to build

Build docker image containing all the tools to generate the website and pdfs
```BASH
git clone https://github.com/cagix/pandoc-lecture.git /tmp/pandoc-lecture
cd  /tmp/pandoc-lecture/docker
make amd64
cd -
```

Mount this dir into build container
```BASH
# the last argument  is the docker image to be started
docker run  --rm -it  -v "$(pwd):/pandoc" -w "/pandoc"  -u "$(id -u):$(id -g)"  --entrypoint "bash"  pandoc-lecture
make slides
...
```
