Folder containing the ontology.

This template uses `ontology/src/config.md` (YAML front matter) to declare:

- `main_ontology`: ontology entrypoint file (relative path)
- `reasoner`: the reasoner to use in workflows

You can use any RDF serialization, but WIDOCO expects RDF/XML files (`.owl` or `.rdf`) and, second best, `.ttl`.
