# KGS03 Unsafe JSON construction in authorization request

## What is happening

The root cause is in `UserAuthClient.cc`. The authorization request body is constructed by manual string concatenation, inserting the `token` and `topic` values directly into a JSON string literal:

```
const std::string reqBody{
    R"({"tk": ")" + token + R"(", "topic": ")" + topic + "\""}"}
};
```

Neither `token` nor `topic` is sanitized or escaped before insertion. 
Both values originate from the client. If either contains characters that have meaning in JSON, quotes, braces, backslashes, commas — the resulting string is no longer the intended object.

## What a crafted value produces

A `token` value of:

```
abc", "extra":"injected
```

produces the request body:

```
{"tk": "abc", "extra":"injected", "topic": "..."}
```

The authentication service now receives a field it did not expect. Depending on how the service parses this, the outcomes range from a silent parse error and request rejection, to the injected field being read and acted on. If the service uses a lenient parser that takes the last occurrence of a duplicate key, an injected `tk` field after the real one could substitute a different token value entirely.

A `topic` value containing a closing brace or quote can truncate the object prematurely, producing malformed JSON that may be rejected or partially parsed.

## Why this matters despite being low severity

The authentication service sits between the GuiServer and the backend. The GuiServer is supposed to be the gatekeeper; if the request it sends to the auth service can be shaped by client input, the trust relationship between those two components is weaker than it looks. There is no confirmed bypass here, but the actual impact depends entirely on the auth service's parser behavior, which is an external dependency not controlled by this codebase.

## Fix

The request body should be constructed with a JSON serialization library rather than by hand. Using `nlohmann::json`:

```
nlohmann::json body;
body["tk"]    = token;
body["topic"] = topic;
const std::string reqBody = body.dump();
```

`nlohmann::json` escapes quotes, backslashes, and control characters before serializing string values. The structure of the output is determined by the library, not by the content of the input.

A token containing `"` or `}` becomes a safely encoded string value rather than a structural element.
