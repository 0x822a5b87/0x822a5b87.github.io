---
title: Draw Diagrams in Markdown
date: 2024-10-28 11:50:28
tags:
    - markdown
    - md
    - diagrams
---

# Draw Diagrams in Markdown

I have always been a enthusiast for diagrams because I believe illustrations are a more direct and efficient way to elucidate the truths we want to explore. Meanwhile, I don't need a heavy diagraming tool like PowerPoint. I prefer to a  `lightweight`, `cross-platform`,`easy-to-using` tool that can be seamlessly integerated with markdown, and here are several tools build for markdown that I find useful:

- [js-sequence-diagrams](https://bramp.github.io/js-sequence-diagrams/)
- [flowchart.js](http://flowchart.js.org)
- [mermaid](https://mermaid.js.org/#/)

If you are interested in any one of these tools, just read the tutorial to get started on your journey.

## Most Useful Diagrams in Mermaid

### flowchart

```mermaid
---
title: 语法树
---
flowchart TD

subgraph condition1["operaor1 >= operator2"]
	operator1:::high --> operand1["1"]
	operator1 --> operand2["2"]
	
	operator2 --> operand3["3"]
	operator2:::low --> operator1
end

subgraph condition2["operaor1 < operator2"]
	op3["operator1"]:::low
	op4["operator2"]:::high
	operand4["1"]
	operand5["2"]
	operand6["3"]

	op3 --> operand4
	op4 --> operand5
	op4 --> operand6
	op3 --> op4
end

classDef low fill:#90ee90,stroke:#333,stroke-width:4px;
classDef high fill:#ff6347,stroke:#333,stroke-width:4px;
```

## sequenceDiagram

```mermaid
sequenceDiagram
	participant Input
	participant SqlParser
	participant SqlValidator
	participant SqlToRelConverter
	participant Planner
	
	Input -->> SqlParser : string
	SqlParser ->> SqlValidator : SqlNode
	SqlValidator -->> SqlValidator : validate(SqlNode)
	SqlValidator ->> SqlToRelConverter : SqlNode
	SqlToRelConverter -->> SqlToRelConverter : convertQuery(SqlNode)
	NOTE OVER SqlToRelConverter : RexBuilder && SqlNodeToRexConverter convert SqlNode to RexNode
  SqlToRelConverter -->> Planner : RelNode
```

### block-beta

```mermaid
---
title: function stack
---
block-beta
columns 3

block:group1:2
	columns 2
	stack3["stack 3"]:2
	stack2["stack 2"]:2
	stack1["stack 1"]:2
	stack0["stack 0"]:1
    stackPointer:1
end
stackPointer --> stack0
desc1["empty"]:1

block:group2:2
	columns 2
	Local2["..."]:2
	Local1["Local 1"]:2
	Local0["Local 0"]:1
    basePointer
	Function:1
	functionPointer
end
desc2["reserved for locals"]:1
basePointer --> Local0
functionPointer --> Function

block:group3:2
	columns 1
	OtherValue1["..."]:1
	OtherValue2["Other Value 3"]:1
	OtherValue3["Other Value 2"]:1
	OtherValue4["Other Value 1"]:1
end
desc3["pushed before call"]

classDef front 1,fill:#FFCCCC,stroke:#333;
classDef back fill:#969,stroke:#333;
classDef op fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef header fill: #696,color: #fff,font-weight: bold,padding: 10px;

class stack3 op
class stack2 op
class stack1 op
class stack0 op

class Local0 header
class Local2 header
class Local1 header
class Function header

class OtherValue1 back
class OtherValue2 back
class OtherValue3 back
class OtherValue4 back

class basePointer front
class stackPointer front
class functionPointer front
```

