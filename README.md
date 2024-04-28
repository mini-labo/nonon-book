# nonon-book

https://book.nonon.house


### Development

To start a local development environment, use the command below:

```bash
docker run -it --init --rm -v $PWD:/book -p 3000:3000 -p 3001:3001 peaceiris/mdbook serve --hostname 0.0.0.0

#
# then navigate to: http://localhost:3000/
```

