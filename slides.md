---
# try also 'default' to start simple
theme: apple-basic
layout: intro
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: true
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Dataloadery | Co to je? K čemu to používat?

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
background: https://images.unsplash.com/photo-1523878288860-7ad281611901?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1171&q=80
---

<div class="absolute top-10">
  <span class="font-700 text-gray-700">
    Dmitrij Tkačenko | 8.12.2021
  </span>
</div>

<div class="absolute bottom-10 text-gray-900">
  <h1>Field Resolvers a<br>Dataloaders<br>v Qestu</h1>
  <p>Co to je? K čemu to používat?</p>
</div>

---
layout: section
---

# Field Resolvers

---

# Motivace

- dotahujeme relace na základě contextu nadřazeného resolveru
- relace se starají samy o sebe
- strom skládáme postupně
- lazy loading
- nejen na relace - dobré i třeba na deprecated fields, nebo jednoduché fieldy s nějakou logikou

```typescript{all|6-9}
@ObjectType()
class Recipe {
  @Field()
  title: string;

  @Field({ deprecationReason: "Use `title` instead" })
  get name(): string {
    return this.title;
  }
}
```
---
layout: statement
---

# Ukázka z DPD

👉 VSCode

<!-- 
Ukazat pickupPoint -> invoicesList -> rewards (s relaci a bez)
-->

---

InvoiceResolver...
```typescript{all|3|all|13|all}
@Query(() => PaginatedInvoiceResponse)
async invoices(@Args() invoicesArgs: InvoicesArgs, @Ctx() { container }: GraphqlContext): Promise<PaginatedInvoiceResponse> {
    const { items, total } = await container.invoiceService.getAllInvoices(invoicesArgs)

    return {
        items: items.map((i) => mapInvoiceEntityToObjectType(i)),
        total,
    }
}

@Query(() => InvoiceResponse)
async invoice(@Args() { id }: InvoiceArgs, @Ctx() { container }: GraphqlContext): Promise<InvoiceResponse | null> {
    const invoice = await container.invoiceRepository.findOne({ where: { id }, relations: ['rewards', 'pickupPoint'] })

    if (!invoice) {
        throw new BaseApolloError(new InvoiceNotFound(`The invoice with the id of: "${id}" was not found!`, id))
    }

    return mapInvoiceEntityToObjectType(invoice)
}
```

- a rozdíl dotazování do DB

---

# Používat?

- za mě ano
- menší chybovost
- větší přehlednost
- jednoduše se můžeme zbavit nadbytečných dotazů do DB

Diskuze 🗣

---
layout: section
---

# Dataloaders

---
layout: quote 
---

# "DataLoader is a generic utility to be used as part of your application's data fetching layer to provide a simplified and consistent API over various remote data sources such as databases or web services via batching and caching."

@Fb, @leebyron

---
layout: fact
---

# 2,271,547
Stažení týdně

---

# Motivace

Získat unifikované API pro získávání dat z různých nezávislých zdrojů. Zároveň mít vyřešenou optimalizaci.
### batching
- dotahuji seznam stejných **typů hodnot** ze stejné DB v různých částech dotazu
- do vstupu pošlu seznam klíčů (typicky IDs)
- na výstupu Promise se seznamem odpovídajících hodnot
### caching
- dotahuji stejnou **hodnotu** z DB jednou
- identifikace podle klíče
- **cache per request**

obě techniky se navzájem kombinují
- [Video vysvětlení od autora](https://www.youtube.com/watch?v=OQTnXNCDywA)

---

# Ukázka z dokumentace

<div grid="~ cols-3 gap-4">
<div class="col-span-2">

```typescript{all|2-4|5-8|9-19|14-19|15-17|18|all}
const UserType = new GraphQLObjectType({
  name: 'User',
  fields: () => ({
    name: { type: GraphQLString },
    bestFriend: {
      type: UserType,
      resolve: user => userLoader.load(user.bestFriendID)
    },
    friends: {
      args: {
        first: { type: GraphQLInt }
      },
      type: new GraphQLList(UserType),
      resolve: async (user, { first }) => {
        const rows = await queryLoader.load([
          'SELECT toID FROM friends WHERE fromID=? LIMIT ?', user.id, first
        ])
        return rows.map(row => userLoader.load(row.toID))
      }
    }
  })
})
```

</div>

- bez Dataloaderu až 13 DB dotazů
- s Dataloadery max 4

</div>

---
layout: statement
---

# Ukázka z DPD

👉 VSCode

<!-- 
Pickup Point pod invoices
-->

---

```typescript{all|6-10|21|all}
@FieldResolver()
async pickupPoint(@Root() invoice: Invoice, @Ctx() { container }: GraphqlContext): Promise<PickupPointResponse | null> {
    if (!invoice.pickupPointPudoId) {
        return null
    }
    const pp = await container.pickupPointRepository.findOneOrFail({
        where: {
            pudoId: invoice.pickupPointPudoId,
        },
    })
    return mapPickupPointEntityToObjectType(pp)
}

// VS

@FieldResolver()
async pickupPoint(@Root() invoice: Invoice, @Ctx() { loaders }: GraphqlContext): Promise<PickupPointResponse | null> {
    if (!invoice.pickupPointPudoId) {
        return null
    }
    const pp = await loaders.pickupPointByPudo.load(invoice.pickupPointPudoId)
    return mapPickupPointEntityToObjectType(pp)
}
```

---

# Používat?

- za mě nemusíme, je to věc nějaké optimalizace
  
  - popřípadě unifikovaného API na dotažení dat (dá se částečně nahradit pomocí services...)
  
  - je to ale 2v1 - api, optimalizace
- pokud budem dodržovat field resolvery, tak se dá jednoduše doimplementovat
- hybrid mod? Nechat to na pocitu
  - přímočaré a v rámci velkého seznamu?
    
    - pak dataloader
    
    - jinak klasika service/repository

Diskuze 🗣
