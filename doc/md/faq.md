---
id: faq
title: Frequently Asked Questions (FAQ)
sidebar_label: FAQ
---

## Questions

[How to create an entity from a struct `T`?](#how-to-create-an-entity-from-a-struct-t)  
[How to create a struct (or a mutation) level validator?](#how-to-create-a-mutation-level-validator)  
[How to write an audit-log extension?](#how-to-write-an-audit-log-extension)  
[How to write custom predicates?](#how-to-write-custom-predicates)  
[How to add custom predicates to the codegen assets?](#how-to-add-custom-predicates-to-the-codegen-assets)  
[How to define a network address field in PostgreSQL?](#how-to-define-a-network-address-field-in-postgresql)  
[How to customize time fields to type `DATETIME` in MySQL?](#how-to-customize-time-fields-to-type-datetime-in-mysql)  
[How to use a custom generator of IDs?](#how-to-use-a-custom-generator-of-ids)  
[How to use a custom XID globally unique ID?](#how-to-use-a-custom-xid-globally-unique-id)  
[How to define a spatial data type field in MySQL?](#how-to-define-a-spatial-data-type-field-in-mysql)  
[How to extend the generated models?](#how-to-extend-the-generated-models)  
[How to extend the generated builders?](#how-to-extend-the-generated-builders)   
[How to store Protobuf objects in a BLOB column?](#how-to-store-protobuf-objects-in-a-blob-column)

## Answers

#### How to create an entity from a struct `T`?

The different builders don't support the option of setting the entity fields (or edges) from a given struct `T`.
The reason is that there's no way to distinguish between zero/real values when updating the database (for example, `&ent.T{Age: 0, Name: ""}`).
Setting these values, may set incorrect values in the database or update unnecessary columns.

However, the [external template](templates.md) option lets you extend the default code-generation assets by adding custom logic.
For example, in order to generate a method for each of the create-builders, that accepts a struct as an input and configure the builder,
use the following template:

```gotemplate
{{ range $n := $.Nodes }}
    {{ $builder := $n.CreateName }}
    {{ $receiver := receiver $builder }}

    func ({{ $receiver }} *{{ $builder }}) Set{{ $n.Name }}(input *{{ $n.Name }}) *{{ $builder }} {
        {{- range $f := $n.Fields }}
            {{- $setter := print "Set" $f.StructField }}
            {{ $receiver }}.{{ $setter }}(input.{{ $f.StructField }})
        {{- end }}
        return {{ $receiver }}
    }
{{ end }}
```

#### How to create a mutation level validator?

In order to implement a mutation-level validator, you can either use [schema hooks](hooks.md#schema-hooks) for validating
changes applied on one entity type, or use [transaction hooks](transactions.md#hooks) for validating mutations that being
applied on multiple entity types (e.g. a GraphQL mutation). For example:

```go
// A VersionHook is a dummy example for a hook that validates the "version" field
// is incremented by 1 on each update. Note that this is just a dummy example, and
// it doesn't promise consistency in the database.
func VersionHook() ent.Hook {
	type OldSetVersion interface {
		SetVersion(int)
		Version() (int, bool)
		OldVersion(context.Context) (int, error)
	}
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			ver, ok := m.(OldSetVersion)
			if !ok {
				return next.Mutate(ctx, m)
			}
			oldV, err := ver.OldVersion(ctx)
			if err != nil {
				return nil, err
			}
			curV, exists := ver.Version()
			if !exists {
				return nil, fmt.Errorf("version field is required in update mutation")
			}
			if curV != oldV+1 {
				return nil, fmt.Errorf("version field must be incremented by 1")
			}
			// Add an SQL predicate that validates the "version" column is equal
			// to "oldV" (ensure it wasn't changed during the mutation by others).
			return next.Mutate(ctx, m)
		})
	}
}
```

#### How to write an audit-log extension?

The preferred way for writing such an extension is to use [ent.Mixin](schema-mixin.md). Use the `Fields` option for
setting the fields that are shared between all schemas that import the mixed-schema, and use the `Hooks` option for
attaching a mutation-hook for all mutations that are being applied on these schemas. Here's an example, based on a
discussion in the [repository issue-tracker](https://github.com/ent/ent/issues/830):

```go
// AuditMixin implements the ent.Mixin for sharing
// audit-log capabilities with package schemas.
type AuditMixin struct{
	mixin.Schema
}

// Fields of the AuditMixin.
func (AuditMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Immutable().
			Default(time.Now),
		field.Int("created_by").
			Optional(),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
		field.Int("updated_by").
			Optional(),
	}
}

// Hooks of the AuditMixin.
func (AuditMixin) Hooks() []ent.Hook {
	return []ent.Hook{
		hooks.AuditHook,
	}
}

// A AuditHook is an example for audit-log hook.
func AuditHook(next ent.Mutator) ent.Mutator {
	// AuditLogger wraps the methods that are shared between all mutations of
	// schemas that embed the AuditLog mixin. The variable "exists" is true, if
	// the field already exists in the mutation (e.g. was set by a different hook).
	type AuditLogger interface {
		SetCreatedAt(time.Time)
		CreatedAt() (value time.Time, exists bool)
		SetCreatedBy(int)
		CreatedBy() (id int, exists bool)
		SetUpdatedAt(time.Time)
		UpdatedAt() (value time.Time, exists bool)
		SetUpdatedBy(int)
		UpdatedBy() (id int, exists bool)
	}
	return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
		ml, ok := m.(AuditLogger)
		if !ok {
			return nil, fmt.Errorf("unexpected audit-log call from mutation type %T", m)
		}
		usr, err := viewer.UserFromContext(ctx)
		if err != nil {
			return nil, err
		}
		switch op := m.Op(); {
		case op.Is(ent.OpCreate):
			ml.SetCreatedAt(time.Now())
			if _, exists := ml.CreatedBy(); !exists {
				ml.SetCreatedBy(usr.ID)
			}
		case op.Is(ent.OpUpdateOne | ent.OpUpdate):
			ml.SetUpdatedAt(time.Now())
			if _, exists := ml.UpdatedBy(); !exists {
				ml.SetUpdatedBy(usr.ID)
			}
		}
		return next.Mutate(ctx, m)
	})
}
```

#### How to write custom predicates?

Users can provide custom predicates to apply on the query before it's executed. For example:

```go
pets := client.Pet.
	Query().
	Where(predicate.Pet(func(s *sql.Selector) {
		s.Where(sql.InInts(pet.OwnerColumn, 1, 2, 3))
	})).
	AllX(ctx)

users := client.User.
	Query().
	Where(predicate.User(func(s *sql.Selector) {
		s.Where(sqljson.ValueContains(user.FieldTags, "tag"))
	})).
	AllX(ctx)
```

For more examples, go to the [predicates](predicates.md#custom-predicates) page, or search in the repository
issue-tracker for more advance examples like [issue-842](https://github.com/ent/ent/issues/842#issuecomment-707896368).

#### How to add custom predicates to the codegen assets?

The [template](templates.md) option enables the capability for extending or overriding the default codegen assets.
In order to generate a type-safe predicate for the [example above](#how-to-write-custom-predicates), use the template
option for doing it as follows:

```gotemplate
{{/* A template that adds the "<F>Glob" predicate for all string fields. */}}
{{ define "where/additional/strings" }}
    {{ range $f := $.Fields }}
        {{ if $f.IsString }}
            {{ $func := print $f.StructField "Glob" }}
            // {{ $func }} applies the Glob predicate on the {{ quote $f.Name }} field.
            func {{ $func }}(pattern string) predicate.{{ $.Name }} {
                return predicate.{{ $.Name }}(func(s *sql.Selector) {
                    s.Where(sql.P(func(b *sql.Builder) {
                        b.Ident(s.C({{ $f.Constant }})).WriteString(" glob" ).Arg(pattern)
                    }))
                })
            }
        {{ end }}
    {{ end }}
{{ end }}
```

#### How to define a network address field in PostgreSQL?

The [GoType](schema-fields.md#go-type) and the [SchemaType](schema-fields.md#database-type)
options allow users to define database-specific fields. For example, in order to define a
 [`macaddr`](https://www.postgresql.org/docs/13/datatype-net-types.html#DATATYPE-MACADDR) field, use the following configuration:

```go
func (T) Fields() []ent.Field {
	return []ent.Field{
		field.String("mac").
			GoType(&MAC{}).
			SchemaType(map[string]string{
				dialect.Postgres: "macaddr",
			}).
			Validate(func(s string) error {
				_, err := net.ParseMAC(s)
				return err
			}),
	}
}

// MAC represents a physical hardware address.
type MAC struct {
	net.HardwareAddr
}

// Scan implements the Scanner interface.
func (m *MAC) Scan(value interface{}) (err error) {
	switch v := value.(type) {
	case nil:
	case []byte:
		m.HardwareAddr, err = net.ParseMAC(string(v))
	case string:
		m.HardwareAddr, err = net.ParseMAC(v)
	default:
		err = fmt.Errorf("unexpected type %T", v)
	}
	return
}

// Value implements the driver Valuer interface.
func (m MAC) Value() (driver.Value, error) {
	return m.HardwareAddr.String(), nil
}
```
Note that, if the database doesn't support the `macaddr` type (e.g. SQLite on testing), the field fallback to its
native type (i.e. `string`).

`inet` example:

```go
func (T) Fields() []ent.Field {
    return []ent.Field{
		field.String("ip").
			GoType(&Inet{}).
			SchemaType(map[string]string{
				dialect.Postgres: "inet",
			}).
			Validate(func(s string) error {
				if net.ParseIP(s) == nil {
					return fmt.Errorf("invalid value for ip %q", s)
				}
				return nil
			}),
    }
}

// Inet represents a single IP address
type Inet struct {
    net.IP
}

// Scan implements the Scanner interface
func (i *Inet) Scan(value interface{}) (err error) {
    switch v := value.(type) {
    case nil:
    case []byte:
        if i.IP = net.ParseIP(string(v)); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", s)
        }
    case string:
        if i.IP = net.ParseIP(v); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", s)
        }
    default:
        err = fmt.Errorf("unexpected type %T", v)
    }
    return
}

// Value implements the driver Valuer interface
func (i Inet) Value() (driver.Value, error) {
    return i.IP.String(), nil
}
```

#### How to customize time fields to type `DATETIME` in MySQL?

`Time` fields use the MySQL `TIMESTAMP` type in the schema creation by default, and this type
 has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC (see, [MySQL docs](https://dev.mysql.com/doc/refman/5.6/en/datetime.html)).

In order to customize time fields for a wider range, use the MySQL `DATETIME` as follows:
```go
field.Time("birth_date").
	Optional().
	SchemaType(map[string]string{
		dialect.MySQL: "datetime",
	}),
```

#### How to use a custom generator of IDs?

If you're using a custom ID generator instead of using auto-incrementing IDs in
your database (e.g. Twitter's [Snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)),
you will need to write a custom ID field which automatically calls the generator
on resource creation.

To achieve this, you can either make use of `DefaultFunc` or of schema hooks -
depending on your use case. If the generator does not return an error,
`DefaultFunc` is more concise, whereas setting a hook on resource creation
will allow you to capture errors as well. An example of how to use
`DefaultFunc` can be seen in the section regarding [the ID field](schema-fields.md#id-field).

Here is an example of how to use a custom generator with hooks, taking as an
example [sonyflake](https://github.com/sony/sonyflake).

```go
// BaseMixin to be shared will all different schemas.
type BaseMixin struct {
	mixin.Schema
}

// Fields of the Mixin.
func (BaseMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Uint64("id"),
	}
}

// Hooks of the Mixin.
func (BaseMixin) Hooks() []ent.Hook {
	return []ent.Hook{
		hook.On(IDHook(), ent.OpCreate),
	}
}

func IDHook() ent.Hook {
    sf := sonyflake.NewSonyflake(sonyflage.Settings{})
	type IDSetter interface {
		SetID(uint64)
	}
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			is, ok := m.(IDSetter)
			if !ok {
				return nil, fmt.Errorf("unexpected mutation %T", m)
			}
			id, err := sf.NextID()
			if err != nil {
				return nil, err
			}
			is.SetID(id)
			return next.Mutate(ctx, m)
		})
	}
}

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		// Embed the BaseMixin in the user schema.
		BaseMixin{},
	}
}
```

#### How to use a custom XID globally unique ID?

Package [xid](https://github.com/rs/xid) is a globally unique ID generator library that uses the [Mongo Object ID](https://docs.mongodb.org/manual/reference/object-id/)
algorithm to generate a 12 byte, 20 character ID with no configuration. The xid package comes with [database/sql](https://golang.org/pkg/database/sql) `sql.Scaner` and
`driver.Valuer` interfaces required by Ent for serialization already implemented.

To store a XID in any string field use the [GoType](schema-fields.md#go-type) schema configuration:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/mixin"
	"github.com/rs/xid"
)

// BaseMixin to be shared will all different schemas.
type User struct {
	mixin.Schema
}

// Fields of the Mixin.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").GoType(xid.ID{}).DefaultFunc(xid.New),
		// ...
	}
}
```

Or as a reusable Ent [Mixin](schema-mixin.md) across multiple schema: 
```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/mixin"
	"github.com/rs/xid"
)

// BaseMixin to be shared will all different schemas.
type BaseMixin struct {
	mixin.Schema
}

// Fields of the Mixin.
func (BaseMixin) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").GoType(xid.ID{}).DefaultFunc(xid.New),
		// Additional base fields. Timestamps, ...
	}
}

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		// Embed the BaseMixin in the user schema.
		BaseMixin{},
	}
}
```

#### How to define a spatial data type field in MySQL?

The [GoType](schema-fields.md#go-type) and the [SchemaType](schema-fields.md#database-type)
options allow users to define database-specific fields. For example, in order to define a
[`POINT`](https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html) field, use the following configuration:

```go
// Fields of the Location.
func (Location) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.Other("coords", &Point{}).
			SchemaType(Point{}.SchemaType()),
	}
}
```

```go
package schema

import (
	"database/sql/driver"
	"fmt"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql"
	"github.com/paulmach/orb"
	"github.com/paulmach/orb/encoding/wkb"
)

// A Point consists of (X,Y) or (Lat, Lon) coordinates
// and it is stored in MySQL the POINT spatial data type.
type Point [2]float64

// Scan implements the Scanner interface.
func (p *Point) Scan(value interface{}) error {
	bin, ok := value.([]byte)
	if !ok {
		return fmt.Errorf("invalid binary value for point")
	}
	var op orb.Point
	if err := wkb.Scanner(&op).Scan(bin[4:]); err != nil {
		return err
	}
	p[0], p[1] = op.X(), op.Y()
	return nil
}

// Value implements the driver Valuer interface.
func (p Point) Value() (driver.Value, error) {
	op := orb.Point{p[0], p[1]}
	return wkb.Value(op).Value()
}

// FormatParam implements the sql.ParamFormatter interface to tell the SQL
// builder that the placeholder for a Point parameter needs to be formatted.
func (p Point) FormatParam(placeholder string, info *sql.StmtInfo) string {
	if info.Dialect == dialect.MySQL {
		return "ST_GeomFromWKB(" + placeholder + ")"
	}
	return placeholder
}

// SchemaType defines the schema-type of the Point object.
func (Point) SchemaType() map[string]string {
	return map[string]string{
		dialect.MySQL: "POINT",
	}
}
```

A full example exists in the [example repository](https://github.com/a8m/entspatial).


#### How to extend the generated models?

Ent supports extending generated types (both global types and models) using custom templates. For example, in order to
add additional struct fields or methods to the generated model, we can override the `model/fields/additional` template like in this
[example](https://github.com/ent/ent/blob/dd4792f5b30bdd2db0d9a593a977a54cb3f0c1ce/examples/entcpkg/ent/template/static.tmpl).


If your custom fields/methods require additional imports, you can add those imports using custom templates as well:

```gotemplate
{{- define "import/additional/field_types" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}

{{- define "import/additional/client_dependencies" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}
```

#### How to extend the generated builders?

In case you want to extend the generated client and add dependencies to all different builders under the `ent` package,
you can use the `"config/{fields,options}/*"` templates as follows:

```gotemplate
{{/* A template for adding additional config fields/options. */}}
{{ define "config/fields/httpclient" -}}
	// HTTPClient field added by a test template.
	HTTPClient *http.Client
{{ end }}

{{ define "config/options/httpclient" }}
	// HTTPClient option added by a test template.
	func HTTPClient(hc *http.Client) Option {
		return func(c *config) {
			c.HTTPClient = hc
		}
	}
{{ end }}
```

Then, you can inject this new dependency to your client, and access it in all builders:

```go
func main() {
	client, err := ent.Open(
		"sqlite3",
		"file:ent?mode=memory&cache=shared&_fk=1",
		// Custom config option.
		ent.HTTPClient(http.DefaultClient),
	)
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()
	ctx := context.Background()
	client.User.Use(func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			// Access the injected HTTP client here.
			_ = m.HTTPClient
			return next.Mutate(ctx, m)
		})
	})
	// ...
}
```


#### How to store Protobuf objects in a BLOB column?

:::info
This solution relies on a recent bugfix that is currently available on the `master` branch and
will be released in `v.0.8.0`
:::


Assuming we have a Protobuf message defined:
```protobuf
syntax = "proto3";

package pb;

option go_package = "project/pb";

message Hi {
  string Greeting = 1;
}
```

We add receiver methods to the generated protobuf struct such that it implements [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner)

```go
func (x *Hi) Value() (driver.Value, error) {
	return proto.Marshal(x)
}

func (x *Hi) Scan(src interface{}) error {
	if src == nil {
		return nil
	}
	if b, ok := src.([]byte); ok {
		if err := proto.Unmarshal(b, x); err != nil {
			return err
		}
		return nil
	}
	return fmt.Errorf("unexpected type %T", src)
}
```

We add a new `field.Bytes` to our schema, setting the generated protobuf struct as its underlying `GoType`:

```go
// Fields of the Message.
func (Message) Fields() []ent.Field {
	return []ent.Field{
		field.Bytes("hi").
			GoType(&pb.Hi{}),
	}
}
```

Test that it works:

```go
package main

import (
	"context"
	"project/ent/enttest"
	"project/pb"
	"testing"

	_ "github.com/mattn/go-sqlite3"
	"github.com/stretchr/testify/require"
)

func Test(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	defer client.Close()

	msg := client.Message.Create().
		SetHi(&pb.Hi{
			Greeting: "hello",
		}).
		SaveX(context.TODO())

	ret := client.Message.GetX(context.TODO(), msg.ID)
	require.Equal(t, "hello", ret.Hi.Greeting)
}

```
