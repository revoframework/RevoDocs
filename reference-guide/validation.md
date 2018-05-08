# Validation

## Command validation

Commands will often come from external untrusted sources, such as APIs publicly exposed to the internet. For that reason, it will often be necessary to do some sanity and integrity checks making sure their data are valid before proceeding to their further processing. As this will usually be considered a cross-cutting concern \(like authorization\) rather independent of the business logic processing the command, the framework implements a pre-command validation filter \(which is enabled by default for all commands\). Any incoming command will automatically be validated using the .NET’s integrated `System.ComponentModel.DataAnnotations.Validator`. This makes it easy to apply arbitrary validation rules using the existing validation attributes infrastructure.

Example:

```csharp
public class AddClassifiedAdCommand : ICommand
{
	[Range(0, decimal.PositiveInfinity)]
	public decimal Price { get; set; }

	[Required]
	public string AdvertismentText { get; set; }
	
	[Required]
	public string PhoneNumber { get; set; }
}
```



