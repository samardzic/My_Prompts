
## Generate quarto file


Files: 
1. gtp_result_final.md = source markdown file that is used for qmd generation
2. rep_result.qmd => this would be ourput file = OUTPUT
3. "->code block<-" = content that is intended to be in code block
4. "->table<-" = content that is intended to be in table
5. "->mermaid<-" = content that is intended to be mermaid diagram
5. "->markdown graph<-" = content that is intended to be markdown graph

- Generate qarto *.qmd file for this markdown document
- Quarto is to deliver in all three formats docx, pdf, html
- For qmd, in header keep format configurations for pdf, docx, html. Also add configuration for mermaid diagrams that are going to be generated
- reference-doc: my_template.docx
- use codeblock where possible
- Im especially interested in codeblock implementation in docx
- Code block formatting:
    - for simple code use indentation of 4 spaces
    - for json indentation of 4 spaces
    - for yaml indentation of 2 spaces
    - use moderate color scheme in syntax highligting
    - use moderate syntax highlighting where possible
    - all code samples have tipical/standard codeblock grey background

- do not use jupiter
- use Mermaid diagrams
    - do not show diagrams in raw format, and also do not show raw code for the diagram
    - show diagrams in rendered format where possible 
    - Where not possible to rendered diagrams show them as pictures in document
    - Important - all diagrams must fit in A4 page. Avoid "big diagrams" or separate them in two diagrams
    - Use mermaid formatting using directives (shown here in example):    
        %%{init:{
  		"theme":"green",
		"themeVariables":{
		"lineColor":"orange"
			}		
		}}%%
	- use straight arrow indicator (curve: linear or similar)
	- use professional color palette -> nothing flashy
	- font-size to be 9-11
	- padding to be no les than 20
			     
- Allowed types of mermaid diagrams:
	- pie
	- sequenceDiagrams
	- stateDiagrams
	- graph LR
	- graph TB
	- flow chart TD
	- gitGraph
	- class diagram
	- treeView-beta

- for "markdown graph Use Unicode Box Drawing Characters

Use:
```text
┌ ┐ └ ┘ │ ─ ├ ┤ ┬ ┴ ┼ ▼ ▲ ◄ ►
```

Avoid:
```text
+----+
|    |
+----+
```
