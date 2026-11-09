---
name: api-docs
description: Write and review API documentation in a structured manner, ensuring consistency and adherence to best practices.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated from the Bruno API documentation
---

# Bruno API Documentation Skill

This skill guides writing Bruno API documentation in OpenCollection YAML format for the `api/Sleg API/` collection.

## File Structure

### Collection Root
- `opencollection.yml` — Collection configuration file
- Each domain has its own folder (e.g., `Productos/`, `Usuarios/`, `Ordenes de compra/`)
- Each folder contains a `folder.yml` and request files

### Naming Conventions
- **Request files**: Pascal case with spaces (e.g., `Create Account.yml`, `List all Products.yml`)
- **Folder files**: Always `folder.yml`
- **Spanish names**: Use Spanish for folder names when they represent business domains (e.g., `Ordenes de compra`, `Usuarios`)

## Templates

### Basic GET Request
```yaml
info:
  name: [Descriptive Name]
  type: http
  seq: [number]

http:
  method: GET
  url: 127.0.0.1:3000/[endpoint]
  auth: inherit

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
```

**Example**: `Productos/List specific Product.yml`

### GET with Query Parameters
```yaml
info:
  name: [Descriptive Name]
  type: http
  seq: [number]

http:
  method: GET
  url: 127.0.0.1:3000/[endpoint]?param1=value1&param2=value2
  params:
    - name: param1
      value: "value1"
      type: query
    - name: param2
      value: "value2"
      type: query
  auth: inherit

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
```

**Example**: `Productos/List all Products.yml`

### POST with JSON Body
```yaml
info:
  name: [Descriptive Name]
  type: http
  seq: [number]

http:
  method: POST
  url: 127.0.0.1:3000/[endpoint]
  body:
    type: json
    data: |-
      {
        "field1": "value1",
        "field2": "value2"
      }
  auth: inherit

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
```

**Examples**: 
- `Autorización/Login.yml`
- `Autorización/Create Account.yml`
- `Productos/Protected/Create Product.yml`

### PATCH/PUT with JSON Body
```yaml
info:
  name: [Descriptive Name]
  type: http
  seq: [number]

http:
  method: PATCH
  url: 127.0.0.1:3000/[endpoint]/[id]
  body:
    type: json
    data: |-
      {
        "field_to_update": "new_value"
      }
  auth: inherit

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
```

**Example**: `Productos/Protected/Update Product.yml`

### Folder Configuration
```yaml
info:
  name: [Folder Name]
  type: folder
  seq: [number]

request:
  auth: inherit
```

**Examples**: All `folder.yml` files in the collection

## Authentication Patterns

### Inherit Auth (Most Common)
Use `auth: inherit` when the request should use authentication from a parent folder or environment.

```yaml
http:
  auth: inherit
```

### Direct Bearer Token (For Specific Endpoints)
Use direct bearer tokens when testing specific protected endpoints with known tokens:

```yaml
http:
  auth:
    type: bearer
    token: ...
```

**Examples**: 
- `Ordenes de compra/Create Order.yml`
- `Reportes/Protected/Dashboard.yml`

### Protected Folder Pattern
Create a `Protected/` subfolder for endpoints requiring authentication:

```
Domain/
├── folder.yml              # request.auth: inherit
├── Public Request.yml      # auth: inherit
└── Protected/
    ├── folder.yml          # request.auth: inherit (or direct auth)
    └── Protected Request.yml  # auth: inherit
```

**Examples**: 
- `Productos/Protected/`
- `Autorización/Protected/`
- `Reportes/Protected/`

## Settings Block

All requests must include the standard settings block:

```yaml
settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
```

This is the conventional configuration for this collection.

## Common Patterns in This Collection

### Pagination Parameters
Use `i` (offset) and `e` (limit) for pagination:

```yaml
params:
  - name: i
    value: "0"
    type: query
  - name: e
    value: "10"
    type: query
```

### Nested JSON Bodies
Complex nested structures are common (see `Ordenes de compra/Create Order.yml`):

```yaml
body:
  type: json
  data: |-
    {
      "contact": {
        "email": "user@example.com",
        "firstName": "John",
        "lastName": "Doe"
      },
      "items": [
        { 
          "variant_id": "uuid", 
          "quantity": 1
        }
      ]
    }
```

### UUID in URLs
Resource IDs are UUIDs in path segments:

```yaml
url: 127.0.0.1:3000/products/9dc3913a-8a5a-4f08-83de-39fa8e70a237
```

## Reference Documentation

Detailed YAML structure reference is available in:
- `references/bruno-yaml-structure-reference.md` — Complete YAML structure documentation
- `references/bruno-opencollection-overview.md` — OpenCollection format overview
- `references/bruno-yaml-samples.md` — Additional YAML samples and patterns

## Checklist for New Requests

1. ✅ Use descriptive Pascal case name with spaces
2. ✅ Set appropriate `seq` number for UI ordering
3. ✅ Use `127.0.0.1:3000` as base URL
4. ✅ Set `auth: inherit` unless testing specific protected endpoint
5. ✅ Include standard `settings` block
6. ✅ Format JSON bodies with literal block scalar (`|-`)
7. ✅ Include all query/path parameters in `params` array
8. ✅ Place in correct domain folder
9. ✅ Create `Protected/` subfolder if authentication is required
10. ✅ Update parent `folder.yml` `seq` if adding new top-level folder
