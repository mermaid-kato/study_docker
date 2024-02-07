```
  web:
    build: .
    depends_on:
      - db
    environment:
      DATABASE_PASSWORD: password
    ports:
      - "3000:3000"
    volumes:
      - .:/docker_rails_test
```
