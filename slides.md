---
theme: seriph
class: text-center
highlighter: shiki
lineNumbers: false
title: Understanding Language Server Protocol
---

# Understanding Language Server Protocol - autocomplete, formatting

<!--

Part of Go's brilliant developer experience is the integration of gopls with text editors like VS Code, Neovim to provide features.
Text editors use the Language Server Protocol standard to communicate.
This standardisation allows multiple text editors to benefit from a single implementation.
In this session, we'll go deeper to find out what's being passed between text editors and the language server, how we can create our own LSPs with Go, and how a project is building on top of the gopls LSP to add autocomplete features to HTML templates.

-->

---

# The protocol 

> The Language Server Protocol (LSP) defines the protocol used between an editor or IDE and a language server that provides language features like auto complete, go to definition, find all references etc.

https://microsoft.github.io/language-server-protocol/

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
