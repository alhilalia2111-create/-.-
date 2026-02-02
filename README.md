# -.-using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

// Single-file minimal web app for .NET 8
// Create project: dotnet new web -o RxSingleFile
// Replace Program.cs with this file, then: dotnet run

var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Logging.AddConsole();

// Configure JSON (camelCase)
builder.Services.ConfigureHttpJsonOptions(opts =>
{
    opts.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    opts.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});

// Allow CORS (simple for local demo)
builder.Services.AddCors();

var app = builder.Build();
app.UseCors(b => b.AllowAnyHeader().AllowAnyMethod().AllowAnyOrigin());

// Seed data
app.Logger.LogInformation("Seeding 1000 platforms...");
var platforms = SeedPlatforms(1000);
app.Logger.LogInformation("Seed completed. Total: {count}", platforms.Count);

// Serve embedded HTML UI at root
app.MapGet("/", async (HttpContext ctx) =>
{
    ctx.Response.ContentType = "text/html; charset=utf-8";
    await ctx.Response.WriteAsync(HtmlPage);
});

// API: list platforms with search, category and paging
app.MapGet("/api/platforms", (HttpRequest req) =>
{
    var q = req.Query;
    string? search = q.ContainsKey("search") && !string.IsNullOrWhiteSpace(q["search"]) ? q["search"].ToString().Trim().ToLowerInvariant() : null;
    string? category = q.ContainsKey("category") && !string.IsNullOrWhiteSpace(q["category"]) ? q["category"].ToString().Trim() : null;
    int page = q.ContainsKey("page") && int.TryParse(q["page"], out var pVal) ? Math.Clamp(pVal, 1, 1000) : 1;
    int pageSize = q.ContainsKey("pageSize") && int.TryParse(q["pageSize"], out var psVal) ? Math.Clamp(psVal, 1, 200) : 50;

    var query = platforms.AsEnumerable();

    if (!string.IsNullOrWhiteSpace(search))
    {
        query = query.Where(p =>
            (!string.IsNullOrEmpty(p.NameAr) && p.NameAr.Contains(search, StringComparison.OrdinalIgnoreCase)) ||
            (!string.IsNullOrEmpty(p.NameEn) && p.NameEn.Contains(search, StringComparison.OrdinalIgnoreCase)) ||
            (!string.IsNullOrEmpty(p.Category) && p.Category.Contains(search, StringComparison.OrdinalIgnoreCase))
        );
    }

    if (!string.IsNullOrWhiteSpace(category) && category != "الكل")
    {
        query = query.Where(p => string.Equals(p.Category, category, StringComparison.OrdinalIgnoreCase));
    }

    var total = query.Count();
    var items = query
        .OrderBy(p => p.NameAr)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new
        {
            p.Id,
            p.NameAr,
            p.NameEn,
            p.Url,
            p.Category,
            p.Icon,
            Services = p.Services
        })
        .ToList();

    return Results.Json(new { total, page, pageSize, items });
});

// API: get by id
app.MapGet("/api/platforms/{id:guid}", (Guid id) =>
{
    var p = platforms.FirstOrDefault(x => x.Id == id);
    return p is null ? Results.NotFound() : Results.Json(p);
});

// Run on predictable port
var url = "http://localhost:5000";
app.Urls.Clear();
app.Urls.Add(url);

app.Logger.LogInformation("Starting Rx single-file app on {url}", url);
app.Run();


// ---------------- Models & Seed implementation ----------------

List<Platform> SeedPlatforms(int total)
{
    var list = new List<Platform>();

    // Fixed known platforms (Arabic + English)
    list.Add(new Platform("أبشر", "Absher", "https://www.absher.sa", "سيادية", "fa-user-shield",
        new[] { "أبشر أفراد", "أبشر أعمال", "خدمات التأشيرات", "تجديد الهوية" }));
    list.Add(new Platform("توكلنا", "Tawakkalna", "https://tawakkalna.sdaia.gov.sa", "سيادية", "fa-shield-alt",
        new[] { "المحفظة الرقمية", "خدمات الفعاليات", "تصاريح العمرة" }));
    list.Add(new Platform("ناجز", "Najiz", "https://najiz.sa", "عدل", "fa-gavel",
ضاء", "التنفيذ", "التراخيص العدلية" }));
    list.Add(new Platform("قوى", "Qiwa", "https://www.qiwa.sa", "عمل", "fa-users-cog",
        new[] { "عقود العمل", "التأشيرات المهنية", "نقل الخدمات" }));
    list.Add(new Platform("مدرستي", "Madrasati", "https://schools.madrasati.sa", "تعليم", "fa-laptop-code",
        new[] { "الاختبارات المركزية", "الفصول الافتراضية", "بنك الأسئلة" }));
    list new Random(12345);

    for (int i = list.Count; i < total; i++)
    {
        var idx = i + 1;
        var cat = categories[rnd.Next(categories.Length)];
        var icon = icons[rnd.Next(icons.Length)];
        var p = new Platform(
            nameAr: $"منصة خدمية رقم {idx}",
            nameEn: $"ServicePlatform {idx}",
            url: "https://www.saudi.gov.sa",
            category: cat,
            icon: icon
        );

        // create 2..6 services
        int svcCount = rnd.Next(2, 7);
        for (int s = 1; s <= svcCount; s++)
        {
            p.Services.Add($"خدمة عامة {s} - رقم {idx}");
        }

        list.Add(p);
    }

    return list;
}

record Platform(Guid Id, string NameAr, string NameEn, string Url, string Category, string Icon)
{
    public Platform(string nameAr, string nameEn, string url, string category, string icon) : this(Guid.NewGuid(), nameAr, nameEn, url, category, icon)
    {
    }

    public List<string> Services { get; init; } = new();
}

// ---------------- Embedded HTML UI (RTL Arabic) ----------------

const string HtmlPage = @"
<!doctype html>
<html lang='ar' dir='rtl'>
<head>
<meta charset='utf-8'>
<meta name='viewport' content='width=device-width, initial-scale=1'>
<title>Rx Platforms - قائمة الخدمات</title>
<link href='https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css' rel='stylesheet'>
<link rel='stylesheet' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css'>
<style>
  body { background:#050608; color:#fff; font-family: 'Cairo', sans-serif; }
  .card { background:#0f1115; border:1px solid rgba(255,255,255,0.03); padding:1rem; border-radius:12px; }
  .cat-btn.active { background: #00f2ff; color:#000; font-weight:800; }
  .scroll-area { height: calc(100vh - 220px); overflow:auto; }
</style>
</head>
<body class='p-6'>
  <header class='flex items-center justify-between gap-4 mb-6'>
    <div class='flex items-center gap-4'>
      <div class='p-3 bg-cyan-400 rounded-xl text-black'><i class='fas fa-terminal'></i></div>
      <div>
        <h1 class='text-2xl font-black'>Rx COMPUTE <span class='text-white'>OS</span></h1>
        <p class='text-sm text-gray-400'>Unified Digital Platform - Demo</p>
      </div>
    </div>

    <div style='width:480px' class='relative'>
      <input id='searchBox' placeholder='ابحث عن منصة أو خدمة...' class='w-full rounded-lg py-3 px-4 bg-black/40 text-sm outline-none' />
      <i class='fas fa-search absolute left-3 top-3 text-gray-400'></i>
    </div>
  </header>

  <section class='mb-4'>
    <div id='categories' class='flex gap-2 overflow-x-auto pb-2'></div>
  </section>

  <main>
    <div class='grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 scroll-area' id='grid'></div>
  </main>

  <footer class='fixed bottom-0 left-0 right-0 bg-black/80 border-t border-white/5 p-3 text-xs flex justify-between'>
    <div><span id='count'>0</span> خدمة ومنصة</div>
    <div>PRODUCED BY Rx TECHNOLOGY © 2026</div>
  </footer>

<script>
const categories = ['الكل','سيادية','تعليم','صحة','عمل','تجارة','عدل','بلدية','أخرى'];
let currentCategory = 'الكل';
let currentSearch = '';
let page = 1;
const pageSize = 50;

const categoriesContainer = document.getElementById('categories');
const grid = document.getElementById('grid');
const countEl = document.getElementById('count');
const searchBox = document.getElementById('searchBox');

function renderCategories(){
  categoriesContainer.innerHTML = categories.map(c => `
    <button class='cat-btn px-4 py-2 rounded-lg border border-white/5 text-sm' data-cat='${c}'>${c}</button>
  `).join('');
  categoriesContainer.querySelectorAll('button').forEach(btn => {
    btn.addEventListener('click', () => {
      categoriesContainer.querySelectorAll('button').forEach(b=>b.classList.remove('active'));
      btn.classList.add('active');
      currentCategory = btn.getAttribute('data-cat');
      page = 1;
      load();
    });
  });
  // set first active
  categoriesContainer.querySelector('button').classList.add('active');
}

async function load(){
  const params = new URLSearchParams();
  params.set('page', String(page));
  params.set('pageSize', String(pageSize));
  if(currentSearch) params.set('search', currentSearch);
  if(currentCategory && currentCategory !== 'الكل') params.set('category', currentCategory);

  const res = await fetch('/api/platforms?' + params.toString());
  const data = await res.json();
  countEl.textContent = data.total;
  renderGrid(data.items);
}

function renderGrid(items){
  grid.innerHTML = items.map(p => `
    <div class='card'>
      <div class='flex justify-between items-start mb-2'>
        <div class='flex items-center gap-3'>
          <div class='w-12 h-12 bg-white/5 rounded-xl flex items-center justify-center text-cyan-300 border border-white/5'>
            <i class='fas ${'${''}'}${'${''}'}'></i>
            <i class='fas ${p.icon}'></i>
          </div>
          <div>
            <div class='text-sm font-black truncate'>${p.nameAr}</div>
            <div class='text-xs text-gray-400'>${p.category}</div>
          </div>
        </div>
        <a href='${p.url}' target='_blank' class='text-cyan-300'><i class='fas fa-external-link-alt'></i></a>
      </div>
      <div class='mt-3 text-xs text-gray-300'>
        <div class='mb-2'>الخدمات:</div>
        <ul class='list-disc list-inside'>
          ${p.services.slice(0,4).map(s => `<li>${s}</li>`).join('')}
        </ul>
      </div>
    </div>
  `).join('');
}

// search handler (debounce)
let debounceTimer = null;
searchBox.addEventListener('input', (e) => {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    currentSearch = e.target.value.trim();
    page = 1;
    load();
  }, 300);
});

renderCategories();
load();
</script>
</body>
</html>