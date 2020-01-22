# endpoint browser links

get properties (or inverse properties) and object classes of an instance

## Parameters

* `endpoint`
  * default: https://tools.jpostdb.org/proxy/sparql
  * example: https://tools.jpostdb.org/proxy/sparql
* `entry`
  * default: http://rdf.jpostdb.org/entry/PRT81_1_Q9NYF8
  * example: http://rdf.jpostdb.org/entry/PRT81_1_Q9NYF8
* `limit`
  * default: 50
* `inv`
  * default: 0
* `bnode`
  * default: 0
* `b_p`
  * example: ["http://rdf.jpostdb.org/ontology/jpost.owl#hasPeptideEvidence"]
* `b_t`
  * example: http://rdf.jpostdb.org/ontology/jpost.owl#PeptideEvidence

## `add_code`

```javascript
({entry, limit, inv, bnode, b_p, b_t})=>{
  inv = parseInt(inv);
  let code = "";
  if(entry.match(/^https*:\/\/.+/) || entry.match(/^urn:\w+:/) || entry.match(/^ftp:/) || entry.match(/^mailto:/)) entry = "<" + entry + ">";
  else if(!entry.match(/^["'].+["']$/) && !entry.match(/^["'].+["']@\w+$/)) entry = '"' + entry + '"';
  if(parseInt(bnode) == 0){
    code = "  VALUES ?s { " + entry + " }\n";
    if(inv == 0) code += "  ?s ?p ?o .";
    else code += "  ?o ?p ?s ." // inverse 
  }else{
    let links = JSON.parse(b_p);
    code = "  {\n    SELECT ?s\n    WHERE {\n";
    let predicates = "<" + links.join(">/<") + ">";
    if(inv) predicates = "<" + links.reverse().join(">/<") + ">";  // inverse
    predicates = predicates.replace(/\<inv-http/g, "^<http"); 
    code += "      " + entry + " " + predicates + " ?s .\n";
    if(b_t) code += "      ?s a <" + b_t + "> .\n";
    code += "    } LIMIT 1\n  }\n";
    if(inv == 0) code += "  ?s ?p ?o .";
    else code += "  ?o ?p ?s ." // inverse 
  }
  let limit_code = "LIMIT " + limit;
  if(limit == 0) limit_code  = "";
  return {subject: code, limit: limit_code};
};
```

## Endpoint

{{endpoint}}

## `links`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT DISTINCT ?s ?p ?c ?c_label (SAMPLE(?o) AS ?o_sample) (COUNT(?o) AS ?o_count)
WHERE {
{{add_code.subject}}
  OPTIONAL {
    ?o a ?c .
    OPTIONAL {
      ?c rdfs:label ?c_label .
    }
  }
  OPTIONAL {
    ?o rdfs:label ?o_label .
  }
}
{{add_code.limit}}
```

## `types`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT DISTINCT ?s ?o ?o_label (COUNT(?x) AS ?c)
WHERE {
{{add_code.subject}}
  ?s a ?o .
  ?x a ?o .
  OPTIONAL {
    ?o rdfs:label ?o_label .
  }
}
ORDER BY DESC(?c)
LIMIT {{limit}}
```

## `return`

```javascript
({links, types, inv})=>{
  let res = [];
  let type_p = {type: "uri", value: "http://www.w3.org/1999/02/22-rdf-syntax-ns#type"};
  let type_o_count = {type: "typed-literal", datatype: "http://www.w3.org/2001/XMLSchema#integer", value: "1"};
  let json = types.results.bindings;
  for(elm of json){
    let obj = {s: elm.s, p: type_p, o_sample: elm.o, c: elm.o, o_count: type_o_count};
    if(elm.o_label) obj['c_label'] = elm.o_label;
    res.push(obj);
  }
  json = links.results.bindings;
  for(elm of json){
    if(elm.p.value != "http://www.w3.org/1999/02/22-rdf-syntax-ns#type") res.push(elm);
  }
  if(inv == 1) inv = true;
  else inv = false;
  return {data: res, inv: inv};
};
```