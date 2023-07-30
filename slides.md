---
theme: seriph
class: text-center
highlighter: shiki
lineNumbers: false
title: Understanding Language Server Protocol
---

# Understanding Language Server Protocol - autocomplete, formatting

---

# Language Server Protocol (LSP)

> ... defines the protocol used between an editor or IDE and a language server that provides language features like auto complete, go to definition, find all references etc.

https://microsoft.github.io/language-server-protocol/

---

# Autocomplete

<img src="autocomplete.gif"/>

---

# Go to definition

<img src="gotodefinition.gif"/>

---

# Diagnostics

<img src="diagnostics.gif"/>

---

# Formatting

<img src="formatting.gif"/>

---

# One protocol for all languages

```mermaid
flowchart LR
    vsc[VS Code] --> lspp
    nvim[Neovim] --> lspp[Language Server Protocol]
    jb[JetBrains] --> lspp
    oe[Other editors...] --> lspp
    lspp --> gp[Go - gopls]
    lspp --> tss[TypeScript - tsserver]
    lspp --> jdtls[Java - jdtls]
    lspp --> other[Other languages...]
```

---

<img src="lsp.jpg"/>

---
layout: section
---

# Stdin / Stdout

---

# Stdin

```sh
ls /
```

---

# Stdout

```sh
ls /
```

```
Applications System       Volumes      cores        etc          nix          
private      sbin         usr          Library      Users        bin
dev          home         opt          run          tmp          var
```

---

# You can send data to a program over stdin

```sh
echo "I drank a cup of tea." | wc -w
```

```
6
```

---

# The data can be anything

```sh
cat public/diagnostics.gif | shasum -a 256
```

```
f83f8b9706f64f550782d61063e21adca0a79162817dc7fd24686d6dc8a95bc4  -
```

---

# Including structured data

```sh
echo '{ "drankItem": { "name": "Earl Grey", "qty": 1 } }' | jq
```

```json
{
  "drankItem": {
    "name": "Earl Grey",
    "qty": 1
  }
}
```

---

# `os.Stdin` is an `io.Reader`, `os.Stdout` is an `io.Writer`

```go {0-3|5|6-10|11-13}
func main() {
	formatJSON(os.Stdout, os.Stdin)
}

func formatJSON(w io.Writer, r io.Reader) (err error) {
	var m map[string]any
	err = json.NewDecoder(r).Decode(&m)
	if err != nil {
		return
	}
	enc := json.NewEncoder(w)
	enc.SetIndent("", "	")
	return enc.Encode(m)
}
```

---
layout: section
---

# LSP Message Format

---

# LSP clients can send requests that expect a response

```mermaid
sequenceDiagram
    participant editor as Editor
    participant lsp as Language Server
    editor->>+lsp: request over stdin
    lsp->>-editor: response over stdout
```

---

# Request messages have header and content parts

```json {1-2|3-16|4-5|6|7-15}
Content-Length: 219\r\n
\r\n
{
	"jsonrpc": "2.0",
	"method": "textDocument/declaration",
	"id": 1,
	"params": {
		"textDocument": {
			"uri": "file:///Users/adrian/project/pizza.cook"
		},
		"position": {
			"line": 10,
			"character": 0
		},
	}
}
```

https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#contentPart

---

# Response messages are linked to the request by the ID

```json {1-2|4|5|6-14}
Content-Length: 222\r\n
\r\n
{
	"jsonrpc": "2.0",
	"id": 1,
	"result": [
		{
			"uri": "file:///Users/adrian/project/pizza.cook",
			"range": {
			    "start": { "line": 5, "character": 23 },
			    "end" : { "line": 6, "character": 0 }
			},
		}
	]
}
```
---

# Editors can send notifications to the server

```mermaid
sequenceDiagram
    participant editor as Editor
    participant lsp as Language Server
    editor-->>lsp: notification over stdin
```


---

# Servers can send notifications to the editor

```mermaid
sequenceDiagram
    participant editor as Editor
    participant lsp as Language Server
    lsp-->>editor: notification over stdout
```

---

# Notifications are the same as requests, but without an ID

TODO: Update this to be a realistic notification

```json {1-2|3-16|4-5|6|7-15}
Content-Length: 219\r\n
\r\n
{
	"jsonrpc": "2.0",
	"method": "textDocument/declaration",
	"params": {
		"textDocument": {
			"uri": "file:///Users/adrian/project/pizza.cook"
		},
		"position": {
			"line": 10,
			"character": 0
		},
	}
}
```

---

# Messages are interleaved

```mermaid
sequenceDiagram
    participant editor as Editor
    participant lsp as Language Server
    editor->>+lsp: request 1
    editor-->>lsp: notification
    editor->>+lsp: request 2
    lsp->>-editor: response 2
    lsp-->>editor: notification
    lsp->>-editor: response 1
```


---
layout: two-cols-header
---

# Request messages are defined as TypeScript interfaces

```ts {|6,8,13}
interface Message {
	jsonrpc: string;
}

interface RequestMessage extends Message {
	id: integer | string;
	method: string;
	params?: array | object;
}

interface NotificationMessage extends Message {
	method: string;
	params?: array | object;
}
```

---

# We can use `json.RawMessage` to defer processing

```go
type Message interface {
  IsJSONRPC() bool
}

type Request struct {
  Version string           `json:"jsonrpc"`
  ID      *json.RawMessage `json:"id"`
  Method  string           `json:"method"`
  Params  json.RawMessage  `json:"params"`
}

func (r Request) IsJSONRPC() bool {
  return r.Version == "2.0"
}

func (r Request) IsNotification() bool {
	return r.ID == nil
}
```

---

# Reading a message

```go {|2-5|6-9|10-13|14-16}
func Read(r *bufio.Reader) (req Request, err error) {
	header, err := textproto.NewReader(r).ReadMIMEHeader()
	if err != nil {
		return
	}
	contentLength, err := strconv.ParseInt(header.Get("Content-Length"), 10, 64)
	if err != nil {
		return req, ErrInvalidContentLengthHeader
	}
	err = json.NewDecoder(io.LimitReader(r, contentLength)).Decode(&req)
	if err != nil {
		return
	}
	if !req.IsJSONRPC() {
		return req, ErrInvalidRequest
	}
	return
}
```

---

# Response messages are defined as TypeScript interfaces

```ts
interface ResponseMessage extends Message {
	id: integer | string | null;
	result?: string | number | boolean | array | object | null;
	error?: ResponseError;
}

interface ResponseError {
	code: integer;
	message: string;
	data?: string | number | boolean | array | object | null;
}
```

---

# In Go, TypeScript unions are reduced to `any`

```go
type Response struct {
	ProtocolVersion string           `json:"jsonrpc"`
	ID              *json.RawMessage `json:"id"`
	Result          any              `json:"result"`
	Error           *Error           `json:"error"`
}

func (r Response) IsJSONRPC() bool { return r.ProtocolVersion == "2.0" }

type Error struct {
	Code int64 `json:"code"`
	Message string `json:"message"`
	Data any `json:"data"`
}

func (e *Error) Error() string { return e.Message }
```

---

# Writing a response

```go {|2-5|6-9|10-12|13}
func Write(w *bufio.Writer, resp Message) (err error) {
	body, err := json.Marshal(resp)
	if err != nil {
		return
	}
	headers := fmt.Sprintf("Content-Length: %d\r\n\r\n", len(body))
	if _, err = w.WriteString(headers); err != nil {
		return
	}
	if _, err = w.Write(body); err != nil {
		return
	}
	return w.Flush()
}
```

---

# Initialization

```mermaid
sequenceDiagram
	User->>Editor: open project
	Editor->>LSP: start LSP server
	Editor->>+LSP: send `initialize` request
	LSP->>-Editor: return `initialize` result
	LSP-->>Editor: send `initialized` notification
```

---

# Keep reading messages


```go
func main() {
	for {
		req, err := Read(t.reader)
		if err != nil { 
			return err 
		}
		switch req.Method {
		}
		if req.IsNotification() {
			t.handleNotification(req)
			return
		}
		t.handleRequestResponse(req)
	}
}
```

---

# Handle request/response messages

```go
func (t *Transport) handleRequestResponse(req Request) {
	mh, ok := t.methodHandlers[req.Method]
	if !ok {
		slog.Error("method not found")
		if err := Write(t.writer, NewResponseError(req.ID, ErrMethodNotFound)); err != nil {
			slog.Error("failed to respond", slog.Any("error", err))
		}
		return
	}
	var res Response
	result, err := mh(req.Params)
	if err != nil {
		log.Error("failed to handle", slog.Any("error", err))
		res = NewResponseError(req.ID, err)
	} else {
		res = NewResponse(req.ID, result)
	}
	if err = Write(t.writer, res); err != nil {
		log.Error("failed to respond", slog.Any("error", err))
		t.error(fmt.Errorf("failed to respond: %w", err))
	}
}
```

---

# Opening a file

```mermaid
sequenceDiagram
	Editor->>LSP: send `textDocument/didOpen` notification
	LSP-->>Editor: send `textDocument/publishDiagnostics` notification
```

---

# Editing an opened file

```mermaid
sequenceDiagram
		Editor->>LSP: send `textDocument/didChange` notification
		LSP-->>Editor: send `textDocument/publishDiagnostics` notification
```

---

---
layout: two-cols-header
---

# JSON-RPC

::left::

* Standard for JSON APIs
* Supports methods and notifications
* Both sides can initiate
* Pipelined
   * Receive a notification while waiting for a method response
   * Send a request while receiving a response
* "Transport agnostic"

::right::

```json
{
  "jsonrpc": "2.0",
  "method": "subtract",
  "params": [
    42,
    23
  ],
  "id": 1
}
```

```json
{
  "jsonrpc": "2.0",
  "result": 19,
  "id": 1
}
```

---

# Reading a request

```go
type Request struct {
	ProtocolVersion string           `json:"jsonrpc"`
	ID              *json.RawMessage `json:"id"`
	Method          string           `json:"method"`
	Params          json.RawMessage  `json:"params"`
}

func Read(r *bufio.Reader) (req Request, err error) {
	// Read header.
	header, err := textproto.NewReader(r).ReadMIMEHeader()
	if err != nil {
		return
	}
	contentLength, err := strconv.ParseInt(header.Get("Content-Length"), 10, 64)
	if err != nil {
		return req, ErrInvalidContentLengthHeader
	}
	// Read body.
	err = json.NewDecoder(io.LimitReader(r, contentLength)).Decode(&req)
	return
}
```

<!--

Part of Go's brilliant developer experience is the integration of gopls with text editors like VS Code, Neovim to provide features.
Text editors use the Language Server Protocol standard to communicate.
This standardisation allows multiple text editors to benefit from a single implementation.
In this session, we'll go deeper to find out what's being passed between text editors and the language server, how we can create our own LSPs with Go, and how a project is building on top of the gopls LSP to add autocomplete features to HTML templates.

-->

---
layout: two-cols-header
---

# So, what is it?

::left::

* JSON-RPC messages

> JSON-RPC is a stateless, light-weight remote procedure call (RPC) protocol. Primarily this specification defines several data structures and the rules around their processing. It is transport agnostic in that the concepts can be used within the same process, over sockets, over http, or in many various message passing environments. It uses JSON (RFC 4627) as data format.

> It is designed to be simple!

https://www.jsonrpc.org/specification

::right::

```json
{"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}
```

```json
{"jsonrpc": "2.0", "result": 19, "id": 1}
```

---

# JSON-RPC over stdin/stdout


---

# JSON-RPC batch operations


---

# Starting up the LSP - VS Code

* Make a VS Code extension

---

# Starting up the LSP - Neovim

* Match based on filename / pattern.

---

# Initialize request

* First request sent from the editor to the LSP
* Sets out client capabilities
* Server returns its capabilities

---

# didOpen

> Client support for textDocument/didOpen, textDocument/didChange and textDocument/didClose notifications is mandatory in the protocol and clients can not opt out supporting them. This includes both full and incremental synchronization in the textDocument/didChange notification.

---

# textEdit
				
