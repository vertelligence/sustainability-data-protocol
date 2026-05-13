# sustainability-data-protocol
![Alt text](./sdp-logo-tmp.png)
An open protocol template for sharing and transferring sustainability data across use-cases.

About this repo:
- Registries folder is for public registration of conceptual model types
    - Type refers to the analysis or object type
    - Id refers to the specific ID of an instance of the above object type, defined by user
    - Units refer to public list of common units in analyses and shared data, to avoid repetition.

- Requirement is that each value in a Registry is unique, so it can be a hashed lookup and
  used in a simple match search in plaintext. This allows for a very lightweight sharing
  configuration.

- Data shared in JSON format template can have fields refer to items in the registry
