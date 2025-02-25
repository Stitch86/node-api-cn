<!-- YAML
added: v0.11.4
changes:
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26625
    description: Return the minimum cipher version, instead of a fixed string
      (`'TLSv1/SSLv3'`).
-->

* Returns: {Object}
  * `name` {string} The name of the cipher suite.
  * `version` {string} The minimum TLS protocol version supported by this cipher
    suite.

Returns an object containing information on the negotiated cipher suite.

For example: `{ name: 'AES256-SHA', version: 'TLSv1.2' }`.

See

for more information.

