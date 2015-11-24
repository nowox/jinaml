# Jin<>JAML
An ambitious standalone templating engine for misc applications (C/C++, HTML, LaTeX, ...)

## What is Jin<>JAML?
JinJAML is a standalone command line tools (also available as module) written in [Python 2.7](https://www.python.org/download/releases/2.7/) allowing to populate templates with data issues from YAML files. 

This modules is based on [jinja2](http://jinja.pocoo.org/docs/dev/) and [HiYaPyCo](https://github.com/zerwes/hiyapyco). YAML files are validated using [pyKwalify](https://github.com/Grokzen/pykwalify). 

```
        .-------------------------.
        |                         v
      .---.                     .---.
      |YML| <----------.        |XSD| (Kwalify Syntax)
      '---'            |        '---'
config.default.yml     |   config.schema.yml
                       |                       .---------.
      .---. -----------'        .---.          |         |        .---.
      |YML| <-------------------|JML|--------->| JinJAML |------->| H |
      '---'                     '---'          |         |        '---'
    config.yml              template.h.jml     '---------'      template.h
```

Initial data are stored into [YAML](http://yaml.org/) files. Additionnaly to the `%YAML 1.2` specs, a YAML file can depend on other YAML files. These files can be used as external namespaces for tags replacement or as dependency. With the latter you can choose from two actions: merge or override. 

## A simple example

Here a very basic configuration:
```
# config.default.yml
%YAML 1.2
%TAG  !schema! config.schema.yml
---
foo: 42
bar: 
   - baz
   - qux
...
```

Which can be validated with a schema:
```
# config.default.yml
%YAML 1.2
--- !pykwalify
type: map
mapping: 
   foo:
      desc: Foo is a parameter used in this very configuration.
      type: int
      range: {min: 0; max: 42} 
   bar:
      desc: Bar can be used to list something useful.
      type: seq
      sequence: 
        type: str
        unique: True
...
```

A basic configuration can be written that inherit from the default configuration

```
# config.yml
%YAML 1.2
%TAG  !schema! config.schema.yml
---
<>: # JinJAML magic tag
   super: {file: config.default.yml; method: merge}
   
foo: 12
bar: 
   - quux
   - norf
other: !!int "{{foo}}"
...
```

Here is the template

```
{% datasource 'config.yml' %}
// Header file (other: {{other}})
#ifndef _FOO_H_
#define _FOO_H_
#define FOO_SIZE {{foo}}
{% for b in bar -%}
char *{{ b }};
{%- endfor %}
#endif // _FOO_H_
```

At the end, Jin<>JAML will process the template as follow: 

1. Read the template and check for `datasource`
2. Read the data source which is `config.yml` and extract the JinJAML `<>:` key
3. Validate with the schema `config.schema.yml` because it is linked to a schema
4. Check its dependance `config.default.yml`
5. Validate with the schema `config.schema.yml` because it is linked to a schema
6. Merge the two files together
7. Apply the inner template `{{foo}}`
8. Process the resulting YAML file into a dictionary

The resulting dictionary will be:

```
{'foo':12; 'bar':['baz','qux','quux','norf']; 'other': 12}
```

With this information, the template can be processed:

```
// Header file (other: 12)
#ifndef _FOO_H_
#define _FOO_H_
#define FOO_SIZE 12
char *baz;
char *qux;
char *quux;
char *norf;
#endif // _FOO_H_
```

## A bolder example: The expresso machine

Let's consider we are a company that manufacture expresso machines. Each expresso machine is based on the exact same electronic that comes in various models and each models have a particular configuration. 

The configuration is for example: 

- The available features
- The menu options
- The temperature and the pressure for each coffee brends
- The color code of the capsules

Each *firmware* will be build witht the enabled features for the concerned product. Menu, temperatures and other parameters are embedded into the C/C++ code base. The pressure and temperature are used for the documentation.

The complete configuration could be stored into a database such as sqlite or mongodb, however, it is eaiser here to store these files as plain text. The configuration can be under Git control and diff, compare and merge become very easy. 

So, we have here different files: 

```
# arch.yml
%YAML 1.2
%TAG  !schema! arch.schema.yml
---
brand: Exos
product_name: Expresso Maker
proc: ARM Cortex M0
...
```

```
# products.yml
%YAML 1.2
%TAG !schema! products.schema.yml
---
<>:
   import: {filename: arch.yml; as: arch}

products:   
   22: 
      name: {{ arch.brand }} Galaxy Coffee Maker
      desc: Design coffee maker with large water tank.
      water_tank: 1000 # ml
   23: &mercury
      name: {{ arch.brand }} Mercury Maker
      desc: Compact maker for small espresso only.
      water_tank: 350 # ml
   29: 
      <<: *mercury
      name: {{ products.22.name }} Deluxe Edition
...
```

The description of the coffee capsules is given with the `capsules.yml` file. The limited edition capsules are located into another file to make easier diff across versions. This second file is merged into the main `capsule.yml` with a `<>` directive. 
```
# capsules.yml
---
<>:
   merge: {filename: 'capsules_limited.yml'}
   
capsules:   
   - name: Roma
     temperature: 89
     pressure: 17.5
     color: brown
     id: 0x2918CF
     
     
   - name: Escobar
     temperature: 73
     pressure: 15
     color: purple
     id: 0xACF23D     
```

## Usage

### Jinja Templates

#### Link a data-source to the template

The directive can take place anythere in the template. However, it would be better to place it at the beginning
```
{% source "foo.yaml" %} 
```

Optionally, the source can be imported in a namespace
```
{% source "foo.yaml", foo %}
```

#### Custom Helpers/Filters
Sometime, its necessary to process the data in a more complex manner. Two options are available.

A filter can be added directly from the template
```
{% addfilter "camelcase.py", camelize %}
```

Or the filter can be embedded into the template. It is written in pure python
```
{% addfilter %}
def foo(x): # Foo will be accessible as a filter with the same name
   return hex((x + 42) & 0x123 )
{% endaddfilter %}
```

### YAML extensions

#### Validation scheme
The validation scheme is declared as a YAML tag named `!schema!` located at the beginning of the file. Defining a validation schema is optional, but strongly advised as it can serve as documentation.

```
%YAML 1.2
%TAG !schema! http://path/to/the/schema.yml
---
```

This can also be an absolute or a relative filesystem path
```
%YAML 1.2
%TAG !schema! ./schema.yml
---
```

The format and the version of the schema validation is given as a tag to the YAML content. It can be omitted
```
%YAML 1.2
--- !pykwalify,1.5
```

#### Magic <>
Jin<>JAML will use the information inside the `<>` key to link, merge, apply a template or even define a merge priority across YAML files. This key can be located anyware in the YAML file and it is optional. 

The supported options are: 

Import other YAML under a particular namespace. If the `as` is omitted, the dataset will be imported without namespace. In case of conflicts, the priority is always the local context.

```
---
<>: 
   import: 
      file: 'foo.yml'
      as: myfoo
bar: "The bar value is {{ myfoo.bar_value }}."
```

If no namespace is required, this sugar syntax will work: `import: foo.yml`

Merges:

A YAML file can be merged into another one using the `merge` directive. By default the following rules apply:

- A scalar entry will be overwritten by the local context
- A list is merged while preserving the natural alphabetic (`natsort`) order. 
- The keys of a map entry are kept into a natural alphabetic order. 
- An empty scalar or map has no effect

Fine behavior for certain elements can be defined within the `policy` key:

```
---
<>:
   merge:
      file: 'foo.yml'
      policy: 
         foo:
            bar: our
            baz: their
         list: ignore
         qux: tail
         quux: head
         fubar: sort
         
foo:
   bar: {a: 1, b:2, c:3} 
   baz: {m: 13, n: 14, o: 15}
   qux: [k, b]
   quux: [a, b]
   fubar: [c, e, z, u]
```   
   
```
# foo.yml
---
foo:
   bar: {c:3, d:4, e:5} 
   baz: {o: 15, q: 16, r: 17}
   list: [23, 42]
   qux: [b, c, d]
   quux: [e, f]
   fubar: [a, e, z, t]
   bam: 0x20
```   

The final dataset will be

```
{foo: 
   {bar: {a: 1, b:2, c:3}, 
    baz: {o: 15, q: 16, r: 17}, 
    qux: [k, b, c, d], 
    quux: [e, f, a, b], 
    fubar: [a, c, e, t, u, z], 
    bam: 0x20
    }
}
```

The allowed directives are: 

- `ignore` Not imported
- `our` Our datas mask theirs (default action for scalars)
- `their` Their datas mask ours
- `head` Additionnal data are added at the beginning of the map or the list
- `tail` Additionnal data are added at the end of the map or the list
- `sort` Normal sort
- `natsort` Natural sort (default action for map and list)
- `combine` Combine scalars into a list 

Policies can be applied to a particular type:

```
<>: 
   merge:
      file: 'foo.yml'
      type-policy:
         map: our
         seq: their
         int: ignore
         str: combine
```

Cross-references:

YAML allows references inside a file such as: 

```
---
foo: &foo_anchor
   - 1
   - 2
   - 3
bar: *foo_anchor
```

In Jin<>JAML, anchors can be used across files. They will be resolved prior to any merge or template fill in. 

```
# a.yml
---
foo: &anchor
   - 1
   - 2
```

```
# b.yml
<>: {merge: a.yml}
bar: *anchor
```

#### Templates
Jinja2 tags `{{ }}` can be used inside a YAML file. The current context is seen by the parser: 

```
---
foo: 
   baz: 2
bar: {{ foo.baz + 4 }}
```

Leads to `{foo: {baz: 2}, bar: 6}`. 

The `import` directive allows to make visible another `YAML` file in the current context:


```
# a.yml
---
foo: 42
```

```
# b.yml
---
<>: {import: a.yml}
bar: {{ foo }}
```

Gives `{bar: 42}`

### Command line usage
The unique command `jinjaml` is fairly easy to use. The trivial case is to evaluate a template: 

```
$ jinjaml foo.h.jml > foo.h
```

The data-source can be manually specified: 

```
$ jinjaml --data=foo.yml foo.h.jml > foo.h
```

**Jin<>JAML** can be used as simple YAML object browser: 

```
$ jinjaml -q foo.yml foo.bar.baz
42
```


