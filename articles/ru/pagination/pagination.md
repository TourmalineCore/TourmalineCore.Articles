# ASP.NET Core + React: Создание server-side таблиц с помощью пакетов TourmalineCore

Готовый код проектов можно найти [здесь](https://github.com/TourmalineCore/TourmalineCore.Articles.Examples/tree/master/pagination-example).

# Реализация

## Back

> Для работы потребуется [.NET Core](https://dotnet.microsoft.com/download) версии 3.0 и выше, а также [Visual Studio](https://visualstudio.microsoft.com/ru/).

1. Создаём новый проект ASP.NET Core Web Application версии 3.0 или выше. В качестве шаблона нужно выбрать ASP.NET Core Empty.

2. Добавляем в проект необходимые Nuget пакеты:
    - [**TourmalineCore.AspNetCore.Pagination**](https://github.com/TourmalineCore/TourmalineCore.AspNetCore.Pagination)
    - **Microsoft.EntityFrameworkCore.InMemory**. Необходимо для создания БД. Для демонстрации нам достаточно будет InMemory-базы.

3. Создаём модели, которые будут использоваться в таблице. В данном случае это будет модель **Product**, имеющий one-to-many связь с сущностью **Vendor**.

```csharp
using System.Collections.Generic;

namespace PaginationExample.Models
{
    public class Vendor
    {
        public long Id { get; set; }

        public string Name { get; set; }
    }
}
```

```csharp
using System;

namespace PaginationExample.Models
{
    public class Product
    {
        public long Id { get; set; }

        public string Name { get; set; }

        public DateTime ExpirationDate { get; set; }

        public int Cost { get; set; }

        public long VendorId { get; set; }
        public Vendor Vendor { get; set; }
    }
}
```

Также, для передачи на фронт будем использовать отдельную модель, совмещающую данные **Product** и **Vendor**.
```csharp
using System;

namespace PaginationExample.Models
{
    public class ProductDto
    {
        public long Id { get; set; }

        public string Name { get; set; }

        public DateTime ExpirationDate { get; set; }

        public int Cost { get; set; }

        public string VendorName { get; set; }
    }
}

```

4. Создаём DbContext. В OnModelCreating задаем связь между сущностями.

```csharp
using Microsoft.EntityFrameworkCore;
using PaginationExample.Models;

namespace PaginationExample.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions options)
            : base(options)
        {
        }

        public virtual DbSet<Product> Products { get; set; }
        public virtual DbSet<Vendor> Vendors { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            modelBuilder.ApplyConfigurationsFromAssembly(GetType().Assembly);

            modelBuilder.Entity<Product>()
                .HasOne<Vendor>()
                .WithMany()
                .HasForeignKey(x => x.VendorId);
        }
    }
}
```

5. Для демонстрации работы необходимо будет иметь данные в базе, поэтому имеет сделать сидинг. 

```csharp
using System;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using PaginationExample.Models;

namespace PaginationExample.Data
{
    public class DataSeeder
    {
        public static void InitDb(IApplicationBuilder app)
        {
            using var serviceScope = app.ApplicationServices.CreateScope();
            var context = serviceScope.ServiceProvider.GetRequiredService<AppDbContext>();

            var vendor1 = new Vendor() { Name = "Сыровая Реальность" };
            var vendor2 = new Vendor() { Name = "ООО Т-Мороженое" };

            context.Vendors.AddRange(vendor1, vendor2);

            context.Products.AddRange(
                new Product
                {
                    Name = "Сыр Российский",
                    Cost = 50,
                    ExpirationDate = DateTime.Today,
                    Vendor = vendor1,
                },
                new Product
                {
                    Name = "Сыр Бри",
                    Cost = 150,
                    ExpirationDate = DateTime.Today.AddDays(5),
                    Vendor = vendor1,
                },
                new Product
                {
                    Name = "Мороженое Фруктовый Лёд",
                    Cost = 250,
                    ExpirationDate = DateTime.Today.AddDays(10),
                    Vendor = vendor2,
                }
            );

            context.SaveChanges();
        }
    }
}
```

6. Теперь нам нужно создать класс-наследник PageQueryBase. В нём будет реализована вся логика по работе с выборкой.

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using PaginationExample.Data;
using PaginationExample.Models;
using TourmalineCore.AspNetCore.Pagination;
using TourmalineCore.AspNetCore.Pagination.Extensions;
using TourmalineCore.AspNetCore.Pagination.Models;

namespace PaginationExample.Queries
{
    public class ProductsQuery : PageQueryBase<Product, ProductDto>
    {
        private readonly AppDbContext _context;

        public ProductsQuery(AppDbContext context)
        {
            _context = context;
        }

        // достаем данные с нужной страницы
        public Task<PaginationResult<ProductDto>> GetPageAsync(PaginationParams paginationParams)
        {
            var queryable = _context.Products
              .AsQueryable()
              .AsNoTracking();

            return GetPageByPaginationParamsAsync(
                    queryable,
                    paginationParams
                );
        }

        // включаем необходимые связанные сущности в выборку
        protected override IQueryable<Product> DoIncludes(IQueryable<Product> queryable)
        {
            return queryable
                .Include(x => x.Vendor);
        }

        // фильтруем выборку
        protected override IQueryable<Product> DoFiltration(IQueryable<Product> queryable, ColumnFilter filter)
        {
            // если фильтр пуст, отдаем квери в исходном виде
            if (string.IsNullOrWhiteSpace(filter.Value))
            {
                return queryable;
            }

            // применяем различные правила фильтрации в зависимости от фильтруемого столбца
            return filter.Name switch
                   {
                       nameof(ProductDto.Name) => queryable.Where(x => x.Name.ToLower().Contains(filter.Value)),
                       nameof(ProductDto.ExpirationDate) => queryable.Where(x => x.ExpirationDate.ToString().Contains(filter.Value)),
                       nameof(ProductDto.VendorName) => queryable.Where(x => x.Vendor.Name.ToLower().Contains(filter.Value)),
                       _ => throw new InvalidOperationException($"Unexpected filter name: {filter.Name}"),
                   };
        }

        // сортируем выборку
        protected override IOrderedQueryable<Product> DoOrdering(IQueryable<Product> queryable, string orderBy, ListSortDirection sortDirection)
        {
          // применяем различные правила сортировки в зависимости от сортируемого столбца
            return orderBy switch
                   {
                       nameof(ProductDto.VendorName) => sortDirection == ListSortDirection.Ascending 
                            ? queryable.OrderBy(x => x.Vendor.Name.ToLower())
                            : queryable.OrderByDescending(x => x.Vendor.Name.ToLower()),
                       _ => queryable.OrderBy(orderBy, sortDirection),
                   };
        }

        // конвертируем исходную сущность в объект, возвращаемый с эндпоинта
        protected override Task<List<ProductDto>> Map(List<Product> entities)
        {
            var dtos = entities.Select(x => new ProductDto
                    {
                        Name = x.Name,
                        Cost = x.Cost,
                        ExpirationDate = x.ExpirationDate,
                        VendorName = x.Vendor.Name,
                    }
                )
                .ToList();

            return Task.FromResult(dtos);
        }
    }
}
```

7. Теперь создаем контроллер с эндпоинтом, по которому мы сможем запросить выборку данных.

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using PaginationExample.Models;
using PaginationExample.Queries;
using TourmalineCore.AspNetCore.Pagination.Extensions;
using TourmalineCore.AspNetCore.Pagination.Models;

namespace PaginationExample.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class ProductsController : ControllerBase
    {
        private readonly ProductsQuery _productsQuery;

        public ProductsController(ProductsQuery productsQuery)
        {
            _productsQuery = productsQuery;
        }

        [HttpGet("all")]
        public async Task<PaginationResult<ProductDto>> GetProducts()
        {
            // преобразуем параметры из запроса в объект
            var paginationParams = Request.Query.GetPaginationParams();

            // возвращем соотвествующую выборку данных
            return await _productsQuery.GetPageAsync(paginationParams);
        }
    }
}
```

8. Обновляем Startup.cs.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using PaginationExample.Data;
using PaginationExample.Queries;

namespace PaginationExample
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddDbContext<AppDbContext>(options => options.UseInMemoryDatabase(databaseName: "ApplicationDb"));
            
            services.AddTransient<ProductsQuery>();

            services.AddControllers();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // Настраиваем CORS. Это необходимо, так как два наших приложения технически будут располагаться в разных доменах
            app.UseCors(
                    builder => builder
                        .AllowAnyHeader()
                        .SetIsOriginAllowed(host => true)
                        .AllowCredentials()
                        .AllowAnyMethod()
                );

            app.UseRouting();

            DataSeeder.InitDb(app);

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
}
```

> **Важно**: предполагается, что Back будет запущен на 5000-ом порту. Это порт, присваемый новым проектам в VS по умолчанию. Если вы хотите изменить это значение, то не забудьте внести правки в код во фронтовом проекте. 

Если всё сделано правильно, то при запуске приложения нам станут доступен запрос вроде этого:

```bash
curl --location --request GET 'http://localhost:5000/products/all?draw=2&page=1&pageSize=100&orderBy=name&orderingDirection=desc&filteredByColumns=name,vendorName&filteredByValues=%D0%91%D1%80%D0%B8,%D0%A1%D1%8B%D1%80%D0%BE%D0%B2%D0%B0%D1%8F'
```

В ответ на который мы должны получить:

```json
{
    "draw": 2,
    "list": [
        {
            "id": 0,
            "name": "Сыр Бри",
            "expirationDate": "2021-06-05T00:00:00+05:00",
            "cost": 250,
            "vendorName": "Сыровая Реальность"
        }
    ],
    "totalCount": 1
}
```

## Front

> Для работы потребуется [node.js](https://nodejs.org/en/). 

1. Создаём заготовку React-приложения
```
npx create-react-app pagination-example-front
```

2. Переходим в созданный каталог 
```
cd pagination-example-front
```

3. Устанавливаем пакет [@tourmalinecore/react-table-responsive](https://www.npmjs.com/package/@tourmalinecore/react-table-responsive)
```
npm i @tourmalinecore/react-table-responsive --save
```

4. Перед использованием компонента таблицы необходимо подключить стили. Это можно сделать один раз в файле index.js

**index.js**
```JSX
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import '@tourmalinecore/react-table-responsive/es/index.css';
import '@tourmalinecore/react-tc-modal/es/index.css';
import '@tourmalinecore/react-tc-ui-kit/es/index.css';

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);

```

5. И наконец, мы можем создать страницу

**App.jsx**
```JSX
import React from 'react';
import {ServerTable} from '@tourmalinecore/react-table-responsive';

function App() {
  return (
    <div className="App">
      <ServerTable
        tableId="uniq-table-id"
        // обозначаем столбцы таблицы, свойство accessor должно соотвестовать названию свойства в объекте, полученному с бэка
        columns={[
          {
            Header: 'Название',
            accessor: 'name',
          },
          {
            Header: 'Стоимость',
            accessor: 'cost',
            // отключаем фильтрацию для столбца, которому он не требуется
            disableFilters: true,
            // можем также сделать преобразование данных столбца в более понятный вид
            Cell: ({row}) => `${row.original.cost} р.`,
          },
          {
            Header: 'Производитель',
            accessor: 'vendorName',
          },
          {
            Header: 'Годен до',
            accessor: 'expirationDate',
            disableFilters: true,
            Cell: ({row}) => new Date(row.original.expirationDate).toDateString(),
          }
        ]}
        // набор действий, который доступен для каждой строки
        actions={[
          {
            name: 'show-action',
            show: (row) => true,
            renderIcon: () => <span>!</span>,
            renderText: (row) => `Показать сообщение`,
            onClick: (e, row) => alert(`Это сообщение о товаре "${row.original.name}"`),
          }
        ]}
        order={{
          id: 'name',
          desc: false,
        }}
        language="ru"
        apiHostUrl="http://localhost:5000"
        dataPath="/products/all"
        requestMethod="GET"
      />
    </div>
  );
}

export default App;
```
