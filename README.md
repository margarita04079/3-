// Program.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<IConstellationRestoreService, ConstellationRestoreService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

app.Run();

// DTOs/RestoreRequest.cs
namespace ConstellationApi.DTOs;

public class RestoreRequest
{
    public string? Reference { get; set; }
    public string? Damaged { get; set; }
}

// DTOs/RestoreResponse.cs
namespace ConstellationApi.DTOs;

public class RestoreResponse
{
    public string MissingSymbols { get; set; } = string.Empty;
    public bool IsComplete { get; set; }
}

// Services/IConstellationRestoreService.cs
using ConstellationApi.DTOs;

namespace ConstellationApi.Services;

public interface IConstellationRestoreService
{
    RestoreResponse Restore(string reference, string damaged);
}

// Services/ConstellationRestoreService.cs
using System.Text;
using ConstellationApi.DTOs;

namespace ConstellationApi.Services;

public class ConstellationRestoreService : IConstellationRestoreService
{
    public RestoreResponse Restore(string reference, string damaged)
    {
        var missing = new StringBuilder();

        for (int i = 0; i < reference.Length; i++)
        {
            char refChar = reference[i];
            bool found = false;

            for (int j = 0; j < damaged.Length; j++)
            {
                if (damaged[j] == refChar)
                {
                    found = true;
                    break;
                }
            }

            if (!found)
            {
                missing.Append(refChar);
            }
        }

        var missingArray = missing.ToString().ToCharArray();
        Array.Sort(missingArray);
        string missingSymbols = new string(missingArray);

        return new RestoreResponse
        {
            MissingSymbols = missingSymbols,
            IsComplete = missingSymbols.Length == 0
        };
    }
}

// Controllers/ConstellationsController.cs
using ConstellationApi.DTOs;
using ConstellationApi.Services;
using Microsoft.AspNetCore.Mvc;

namespace ConstellationApi.Controllers;

[ApiController]
[Route("api/constellations")]
public class ConstellationsController : ControllerBase
{
    private readonly IConstellationRestoreService _restoreService;

    public ConstellationsController(IConstellationRestoreService restoreService)
    {
        _restoreService = restoreService;
    }

    [HttpPost("restore")]
    [ProducesResponseType(typeof(RestoreResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public IActionResult Restore([FromBody] RestoreRequest request)
    {
        if (request is null)
        {
            return BadRequest(new { error = "Тело запроса обязательно." });
        }

        if (string.IsNullOrWhiteSpace(request.Reference))
        {
            return BadRequest(new { error = "Поле 'reference' обязательно." });
        }

        if (request.Damaged is null)
        {
            return BadRequest(new { error = "Поле 'damaged' обязательно." });
        }

        if (!IsValidReference(request.Reference))
        {
            return BadRequest(new { error = "Поле 'reference' должно содержать только уникальные заглавные латинские буквы A-Z." });
        }

        if (!IsValidDamaged(request.Damaged, request.Reference))
        {
            return BadRequest(new { error = "Поле 'damaged' должно содержать только символы из 'reference'." });
        }

        var result = _restoreService.Restore(request.Reference, request.Damaged);
        return Ok(result);
    }

    private static bool IsValidReference(string reference)
    {
        if (reference.Length == 0 || reference.Length > 26)
        {
            return false;
        }

        for (int i = 0; i < reference.Length; i++)
        {
            char c = reference[i];
            if (c < 'A' || c > 'Z')
            {
                return false;
            }

            for (int j = i + 1; j < reference.Length; j++)
            {
                if (reference[j] == c)
                {
                    return false;
                }
            }
        }

        return true;
    }

    private static bool IsValidDamaged(string damaged, string reference)
    {
        for (int i = 0; i < damaged.Length; i++)
        {
            char c = damaged[i];
            bool found = false;

            for (int j = 0; j < reference.Length; j++)
            {
                if (reference[j] == c)
                {
                    found = true;
                    break;
                }
            }

            if (!found)
            {
                return false;
            }
        }

        return true;
    }
}
