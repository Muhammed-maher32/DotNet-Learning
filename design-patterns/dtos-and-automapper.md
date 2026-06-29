# DTOs & AutoMapper

> A DTO (Data Transfer Object) is a plain class shaped for a specific job: what a screen needs, or what an API sends/receives. You map between your entities and DTOs so the database model never leaks out to the edges. AutoMapper just automates the boring copying.

---

## Why not just return entities?

Sending your EF entities straight to the view or API is tempting but bites you:

* **Over-exposing data.** Your `User` entity has `PasswordHash`, `IsAdmin`, internal flags. You don't want those serialized out.
* **Over-posting.** If you bind incoming JSON straight onto an entity, a malicious client can set fields you never meant to expose (like `IsAdmin = true`).
* **Coupling.** The screen now depends on the exact shape of your DB table. Change the table, break the UI.
* **Shaping.** A details screen wants the address as one string `"15-Tahrir-Cairo"`, not three columns. A DTO can flatten that.

So you make a DTO per use case:

```csharp
// what the create form sends in
public class CreateMemberDto
{
    public string Name { get; set; } = default!;
    public string Email { get; set; } = default!;
    public string City { get; set; } = default!;
    public string Street { get; set; } = default!;
    public int BuildingNumber { get; set; }
}

// what the details screen needs out (flattened, no internal fields)
public class MemberDetailsDto
{
    public int Id { get; set; }
    public string Name { get; set; } = default!;
    public string Address { get; set; } = default!;  // "15-Tahrir-Cairo"
}
```

> In MVC these are often called ViewModels. Same idea, the name just signals "shaped for a view".

---

## The manual mapping pain

Without a tool you write this by hand, everywhere:

```csharp
var dto = new MemberDetailsDto
{
    Id = member.Id,
    Name = member.Name,
    Address = $"{member.Address.BuildingNumber}-{member.Address.Street}-{member.Address.City}"
};
```

Fine for one. Annoying for fifty. Easy to forget a field. That's what AutoMapper removes.

---

## AutoMapper basics

You define the maps once in a `Profile`:

```csharp
public class MappingProfile : Profile
{
    public MappingProfile()
    {
        // same-named properties copy automatically
        CreateMap<Member, MemberDetailsDto>()
            // only configure the special cases
            .ForMember(dest => dest.Address,
                       opt => opt.MapFrom(src =>
                           $"{src.Address.BuildingNumber}-{src.Address.Street}-{src.Address.City}"));

        // .ReverseMap() makes it work both directions
        CreateMap<CreateMemberDto, Member>().ReverseMap();
    }
}
```

Register it (in `Program.cs`):

```csharp
builder.Services.AddAutoMapper(typeof(MappingProfile));
```

Use it via injected `IMapper`:

```csharp
public class MemberService
{
    private readonly IMapper _mapper;
    public MemberService(IMapper mapper) => _mapper = mapper;

    public MemberDetailsDto Get(Member member)
        => _mapper.Map<MemberDetailsDto>(member);   // one line, all fields handled
}
```

---

## The handy bits

```csharp
// map a whole collection
var dtos = _mapper.Map<IEnumerable<MemberDetailsDto>>(members);

// ignore a property you don't want mapped
CreateMap<CreateMemberDto, Member>()
    .ForMember(d => d.Id, o => o.Ignore());

// map into an EXISTING object (e.g. updating an entity from a dto)
_mapper.Map(updateDto, existingMember);
```

---

## Watch out for

* **It's "magic".** A renamed property silently stops mapping (no error, just null). Profiles can drift from reality. AutoMapper has `AssertConfigurationIsValid()` to catch unmapped members, useful in a test.
* **Don't map heavy logic.** If a mapping needs DB calls or real business rules, that's not mapping anymore. Do it in the service.
* **Some teams skip it** and map by hand on purpose, because it's explicit and greppable. Both are valid. For lots of similar DTOs, AutoMapper wins. For a handful, manual is fine.

---

## Summary

| Thing | Point |
| ----- | ----- |
| DTO / ViewModel | a class shaped for one use case (form in, screen out, API payload) |
| Why | hide internal fields, stop over-posting, decouple UI from DB, reshape data |
| AutoMapper | auto-copies same-named props, you only configure the exceptions |
| `Profile` | where you declare the maps |
| `IMapper.Map<T>(src)` | does the conversion |
| caution | mappings can silently drift, validate them; keep logic out of maps |

> Real example: I used AutoMapper profiles in my GymSystem project to map entities to ViewModels (flattening the address, mapping create forms onto entities).
