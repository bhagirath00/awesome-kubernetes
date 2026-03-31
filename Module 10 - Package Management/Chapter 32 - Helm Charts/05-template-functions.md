# Template Functions

## String Functions

### Case Conversion

```yaml
# Upper case
{{ .Values.appName | upper }}

# Lower case
{{ .Values.appName | lower }}

# Title case
{{ "hello world" | title }}

# Camel case
{{ "hello_world" | camelcase }}
```

### Trimming

```yaml
# Trim quotes
{{ .Values.message | trimAll "\"" }}

# Trim prefix
{{ .Values.domain | trimPrefix "https://" }}

# Trim suffix
{{ .Values.path | trimSuffix "/" }}

# Trim spaces
{{ .Values.text | trim }}
```

### Replacement

```yaml
# Replace substring
{{ .Values.message | replace "hello" "hi" }}

# Replace multiple
{{ .Values.text | replace "a" "1" | replace "e" "2" }}
```

### Quoting

```yaml
# Quote for YAML strings (adds quotes)
password: {{ .Values.secret | quote }}

# Unquote (removes quotes)
{{ .Values.quoted | unquote }}

# HTML escape
{{ .Values.html | htmlEscape }}
```

---

## List Functions

```yaml
# Length
count: {{ len .Values.items }}

# First N items
first: {{ first .Values.items }}
first3: {{ .Values.items | first 3 }}

# Last N items
last: {{ last .Values.items }}
last2: {{ .Values.items | last 2 }}

# Join list with separator
names: {{ join "," .Values.namespaces }}

# Reverse
reversed: {{ reverse .Values.items }}

# Sort
sorted: {{ sort .Values.items }}

# Unique
unique: {{ uniq .Values.items }}

# Without (remove item)
filtered: {{ without .Values.items "unwanted" }}
```

---

## Dictionary Functions

```yaml
# Keys
{{- range keys .Values.config }}
- {{ . }}
{{- end }}

# Values
{{- range values .Values.config }}
- {{ . }}
{{- end }}

# Has key
{{- if hasKey .Values.config "database" }}
Database configured
{{- end }}

# Get value with default
{{ get .Values.config "optional" | default "default-value" }}

# Merge
{{- $merged := merge .Values.global .Values.app }}
```

---

## Math Functions

```yaml
# Addition
sum: {{ add 1 2 3 }}

# Subtraction
result: {{ sub 10 3 }}

# Multiplication
product: {{ mul 5 4 }}

# Division
quotient: {{ div 10 2 }}

# Modulo
remainder: {{ mod 10 3 }}

# Max
maximum: {{ max 5 10 3 }}

# Min
minimum: {{ min 5 10 3 }}
```

---

## Comparison Functions

```yaml
# Equals
{{- if eq .Values.environment "production" }}
Production mode
{{- end }}

# Not equal
{{- if ne .Values.replicas 1 }}
Multiple replicas
{{- end }}

# Greater than
{{- if gt .Values.replicas 3 }}
High replication
{{- end }}

# Greater than or equal
{{- if ge .Values.port 8080 }}
Port >= 8080
{{- end }}

# Less than
{{- if lt .Values.timeout 30 }}
Short timeout
{{- end }}

# Less than or equal
{{- if le .Values.memory 512 }}
Low memory
{{- end }}

# In list
{{- if has "admin" .Values.users }}
Admin role exists
{{- end }}
```

---

## Type Conversion

```yaml
# To string
string: {{ toString 123 }}

# To integer
integer: {{ int "456" }}

# To float
float: {{ float "3.14" }}

# To bool
boolean: {{ bool "true" }}
```

---

## YAML Functions

```yaml
# YAML string to YAML structure
{{- $yaml := .Files.Get "config.yaml" | fromYaml }}
{{- $key := $yaml.database.host }}

# Convert to YAML (for embedding)
config: |
  {{- toYaml .Values.config | nindent 2 }}

# Compact JSON
{{- toJson .Values.data }}

# Pretty print
{{- toPrettyJson .Values.data }}
```

---

## Indentation Functions

```yaml
# nindent: newline + indent
# Add newline, then indent N spaces
spec:
  {{- toYaml .Values.spec | nindent 2 }}

# indent: indent N spaces (no newline)
data:
  config: {{ toYaml .Values.config | indent 4 }}
```

---

## File Functions

```yaml
# Read file from chart
{{- $content := .Files.Get "files/config.txt" }}
data:
  config.txt: |
    {{ $content | nindent 4 }}

# Get multiple files
{{- $files := .Files.Glob "config/**/*.txt" }}
{{- range $path, $_ := $files }}
  {{ base $path }}: {{ .Files.Get $path | quote }}
{{- end }}

# Get file as base64
{{- .Files.Get "secret.txt" | b64enc }}
```

---

## URL Functions

```yaml
# Parse URL
{{- $url := .Values.databaseURL | urlParse }}
host: {{ $url.Host }}
path: {{ $url.Path }}
query: {{ $url.Query }}
```

---

## Crypto Functions

```yaml
# Base64 encode
secret: {{ .Values.password | b64enc }}

# Base64 decode
password: {{ .Values.encodedPassword | b64dec }}

# SHA256 hash
checksum: {{ .Values.data | sha256sum }}

# UUID
id: {{ uuidv4 }}
```

---

## Real-World Examples

### Example 1: Multi-database Support

```yaml
# values.yaml
database:
  type: postgresql  # or mysql, mongodb
  host: db.default.svc
  port: 5432

# template
{{- if eq .Values.database.type "postgresql" }}
DATABASE_DRIVER: postgresql
DATABASE_PORT: "5432"
{{- else if eq .Values.database.type "mysql" }}
DATABASE_DRIVER: mysql
DATABASE_PORT: "3306"
{{- else if eq .Values.database.type "mongodb" }}
DATABASE_DRIVER: mongodb
DATABASE_PORT: "27017"
{{- end }}
```

### Example 2: Dynamic ConfigMap

```yaml
# values.yaml
config:
  LOG_LEVEL: INFO
  DEBUG: "false"
  MAX_CONNECTIONS: "100"

# template
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

### Example 3: Environment-Specific Resources

```yaml
# values.yaml
environment: production
resources:
  dev:
    requests:
      memory: 128Mi
  prod:
    requests:
      memory: 1Gi

# template
resources:
  {{- if eq .Values.environment "production" }}
  {{- toYaml .Values.resources.prod | nindent 2 }}
  {{- else }}
  {{- toYaml .Values.resources.dev | nindent 2 }}
  {{- end }}
```

---

## Best Practices

1. **Use Functions Sparingly**
   - Keep logic simple
   - Complex logic in helpers

2. **Quote for Safety**
   - Always quote strings
   - Prevent YAML parsing issues

3. **Use Proper Indentation**
   - nindent for new lines
   - indent for continuations

4. **Provide Defaults**
   - Use | default for optional values
   - Prevent errors

5. **Test Functions**
   - Use `helm template --debug`
   - Verify output before install

---

## Summary

Template functions:

**Categories:**
- String: upper, lower, trim, replace
- List: len, first, last, join, sort
- Math: add, mul, div, max, min
- YAML: toYaml, fromYaml, toPrettyJson
- Crypto: b64enc, sha256sum, uuidv4
- Indentation: nindent, indent

**Common Patterns:**
- `\{{ value | function \}}`
- `\{{ .Values.key | default "fallback" \}}`
- `\{{ .Values.string | quote \}}`

**Module 10 Chapter 32 Complete - 5 Files!**

You now understand Helm charts:
- File 01: What is a chart
- File 02: Chart structure
- File 03: values.yaml configuration
- File 04: Chart templates and syntax
- File 05: Template functions and filters

**Next:** Chapter 33 covers Helm in practice.
