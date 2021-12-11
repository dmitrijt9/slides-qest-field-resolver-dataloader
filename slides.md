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
download: true
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

* za mě v daný moment nemusíme
  * nemáme zatím více datových zdrojů
  * z FE nejsou moc hluboké dotazy "zatím"
* pokud budem dodržovat field resolvery, tak se dá jednoduše a postupně doimplementovat
* hybrid mod? Nechat to na pocitu, popřípadě na potřebách z FE

Diskuze 🗣

---

Díky 🙏
<div class="space-y-4 ml-2">
<a href="https://github.com/dmitrijt9" class="flex items-center w-min hover:opacity-100 opacity-70">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="1.2em" height="1.2em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M5.883 18.653c-.3-.2-.558-.455-.86-.816a50.32 50.32 0 0 1-.466-.579c-.463-.575-.755-.84-1.057-.949a1 1 0 0 1 .676-1.883c.752.27 1.261.735 1.947 1.588c-.094-.117.34.427.433.539c.19.227.33.365.44.438c.204.137.587.196 1.15.14c.023-.382.094-.753.202-1.095C5.38 15.31 3.7 13.396 3.7 9.64c0-1.24.37-2.356 1.058-3.292c-.218-.894-.185-1.975.302-3.192a1 1 0 0 1 .63-.582c.081-.024.127-.035.208-.047c.803-.123 1.937.17 3.415 1.096A11.731 11.731 0 0 1 12 3.315c.912 0 1.818.104 2.684.308c1.477-.933 2.613-1.226 3.422-1.096c.085.013.157.03.218.05a1 1 0 0 1 .616.58c.487 1.216.52 2.297.302 3.19c.691.936 1.058 2.045 1.058 3.293c0 3.757-1.674 5.665-4.642 6.392c.125.415.19.879.19 1.38a300.492 300.492 0 0 1-.012 2.716a1 1 0 0 1-.019 1.958c-1.139.228-1.983-.532-1.983-1.525l.002-.446l.005-.705c.005-.708.007-1.338.007-1.998c0-.697-.183-1.152-.425-1.36c-.661-.57-.326-1.655.54-1.752c2.967-.333 4.337-1.482 4.337-4.66c0-.955-.312-1.744-.913-2.404a1 1 0 0 1-.19-1.045c.166-.414.237-.957.096-1.614l-.01.003c-.491.139-1.11.44-1.858.949a1 1 0 0 1-.833.135A9.626 9.626 0 0 0 12 5.315c-.89 0-1.772.119-2.592.35a1 1 0 0 1-.83-.134c-.752-.507-1.374-.807-1.868-.947c-.144.653-.073 1.194.092 1.607a1 1 0 0 1-.189 1.045C6.016 7.89 5.7 8.694 5.7 9.64c0 3.172 1.371 4.328 4.322 4.66c.865.097 1.201 1.177.544 1.748c-.192.168-.429.732-.429 1.364v3.15c0 .986-.835 1.725-1.96 1.528a1 1 0 0 1-.04-1.962v-.99c-.91.061-1.662-.088-2.254-.485z" fill="currentColor"></path></svg>
dmitrijt9
</a>
<a href="https://twitter.com/dimot_9" class="flex items-center w-min hover:opacity-100 opacity-70">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="1.2em" height="1.2em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M15.3 5.55a2.9 2.9 0 0 0-2.9 2.847l-.028 1.575a.6.6 0 0 1-.68.583l-1.561-.212c-2.054-.28-4.022-1.226-5.91-2.799c-.598 3.31.57 5.603 3.383 7.372l1.747 1.098a.6.6 0 0 1 .034.993L7.793 18.17c.947.059 1.846.017 2.592-.131c4.718-.942 7.855-4.492 7.855-10.348c0-.478-1.012-2.141-2.94-2.141zm-4.9 2.81a4.9 4.9 0 0 1 8.385-3.355c.711-.005 1.316.175 2.669-.645c-.335 1.64-.5 2.352-1.214 3.331c0 7.642-4.697 11.358-9.463 12.309c-3.268.652-8.02-.419-9.382-1.841c.694-.054 3.514-.357 5.144-1.55C5.16 15.7-.329 12.47 3.278 3.786c1.693 1.977 3.41 3.323 5.15 4.037c1.158.475 1.442.465 1.973.538z" fill="currentColor"></path></svg>
dimot_9
</a>
<a href="https://www.dimot.dev/" class="flex items-center w-min hover:opacity-100 opacity-70">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="1.2em" height="1.2em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M20 22h-2v-2a3 3 0 0 0-3-3H9a3 3 0 0 0-3 3v2H4v-2a5 5 0 0 1 5-5h6a5 5 0 0 1 5 5v2zm-8-9a6 6 0 1 1 0-12a6 6 0 0 1 0 12zm0-2a4 4 0 1 0 0-8a4 4 0 0 0 0 8z" fill="currentColor"></path></svg>
dimot.dev
</a>
</div>

<div class="flex items-center absolute bottom-0 p-8 text-xs opacity-50">
*Pro stažení PDF klikněte na <svg class="mx-2" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="1.2em" height="1.2em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 32 32"><path d="M26 24v4H6v-4H4v4a2 2 0 0 0 2 2h20a2 2 0 0 0 2-2v-4z" fill="currentColor"></path><path d="M26 14l-1.41-1.41L17 20.17V2h-2v18.17l-7.59-7.58L6 14l10 10l10-10z" fill="currentColor"></path></svg> v dolní liště.
</div>