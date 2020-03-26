UI System
==================

The game uses a custom [MVVM (Model-View-ViewModel)](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) system for UI. 

* The ***Model** layer is formed by the data structures in [DataObjects.cs](/Assets/Scripts/DataObjects.cs) and the other helper functions and gameplay state scattered throughout the codebase.
* The **ViewModel** layer sits on top of that, and lives in [ViewModels folder](/Assets/Scripts/ViewModels). These adapt the data from the Model into a format that is tailored for displaying in the UI. For example, the ViewModel layer might take a character's social network and transform that into an integer for how many cities they are connected to, and the UI shows that number in a text object. 
* The **View** layer has two parts:
  * *UGUI components* on the GameObject - define the structure, layout, and style of the UIs, (as XAML would in WPF, or HTML in ASP.NET or javascript MVVM frameworks)
  * *Code-Behind* - which lives in the [Views folder](/Assets/Scripts/Components/Views), are MonoBehaviours that live on the UGUI UI GameObjects. These View classes are the ["code-behind"](https://en.wiktionary.org/wiki/code-behind) for the Views and are mostly just responsible for binding the View's components (such as Buttons, Text, etc) to properties of the ViewModel

# ViewModel-View Binding Example

As an example, say you want to have a CargoItem view that displays the cargo's name and quantity. The game's representation of cargo is the [Resource class](/Assets/Scripts/DataObjects.cs#L586). We'd need to create a View class and a ViewModel class that drives it.

```cs
public class CargoItemViewModel : Model
{
	Resource Cargo;

	// The Name is not expected to change, so we can do a one-way pull from the Model layer
	public string Name => Cargo.name;

	// The Quantity may change externally at any time, and the amount_kg variable has the ability to notify listeners when it changes
	// IValueModel<int> is a wrapper around the Model layer's value that will let us watch for changes and update the UI automatically
	public IValueModel<int> Quantity;

	// The ViewModel will take in the cargo item that we want the View to display (eg. lumber, food, water)
	public CargoItemViewModel(Resource cargo) {
		Cargo = cargo;

		// and now we set up our Quantity to listen to the Model layer's amount_kg value
		Quantity = ValueModel.Wrap(Cargo.amount_kg);
	}
}
```

And then we need to make a View class to bind the UI to the ViewModel data that drives it:

```cs
public class CargoItemView : ViewBehaviour<CargoItemViewModel>
{
	// These are the nested views for the UGUI Text objects, which are assigned in the Unity inspector
	[SerializeField] StringView Name = null;
	[SerializeField] StringView Quantity = null;

	public override void Bind(CargoItemViewModel model) {
		base.Bind(model);

		// The Name text will pull its data from the Name property of our ViewModel
		Name?.Bind(new BoundModel<string>(Model, nameof(Model.Name)));

		// The Quantity text will pull from the Quantity property, but that's an int, so we transform the BoundModel<int> into a BoundModel<string> 
		Quantity?.Bind(new BoundModel<int>(Model, nameof(Model.Quantity)).AsString());
	}
}
```

Then drop that component onto the GameObject and hook up the inspector references.

# Motivation

This MVVM system may be overkill for some very simple UIs, and it's not unreasonable to set up small UIs in a simpler way, but for more complex UIs this can simplify the code a lot. The major benefits are:

* Readability, maintainability
	* Clear separation of the presentation logic (animations, visual state) from the "behavior" of the UI (what data drives it, how it reacts to user input commands)
	* Views are completely decoupled from the game logic and underlying data structure
	  * For example, a view could bind to an interface or abstract model that could have many implementatiions
	* Likewise, the ViewModel is completely independent of the View, so the behavior of a UI can be unit tested without needing the actual front-end View
	* Easy to have the same data drive multtiple views, ie. they are all "views" of the data
* Iteration speed
	* Very easy to make a UI that automatically updates from external state changes (such as the quantity of food and water depleting while you sail)
	  * For example, ```BoundModel```
	* Automatic population of lists, again with automatic updating as items are added/removed
	  * For example, see the ObservableCollection logic in [CityViewModel](/Assets/Scripts/ViewModels/CityViewModel.cs)
	* Lightweight adaptation of model data into whatever format is needed in the UI using a pipeline of data transformation, inspired by [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming)
	  * For example, ```BoundModel.AsString```, or the time formatting transformation in [TimePassingView](/Assets/Scripts/Components/Views/TimePassingView.cs)

# Built-in Views

The game's MVVM system has built in support for these Views, which have their own ViewModel pre-defined. You can use these as "controls" in larger UIs:

* [ButtonView](/Assets/Scripts/_ShinyUnity/UI/Views/ButtonView.cs)
  * Binds to label string and an OnClick action
* [StringView](/Assets/Scripts/_ShinyUnity/UI/Views/StringView.cs)
  * Binds to a string 
* [ImageView](/Assets/Scripts/_ShinyUnity/UI/Views/ImageView.cs)
  * Binds to a Sprite asset
* [ListView](/Assets/Scripts/_ShinyUnity/UI/Views/ListView.cs)
  * Binds to an ObservableCollection, and takes a ViewModel type and a View prefab set in the inspector for its items
  * Automatically handles populating/adding/removing item views when the backing ObservableCollection is changed
* [MessageBoxView](/Assets/Scripts/_ShinyUnity/UI/Views/MessageBoxView.cs)
  * Binds to a title and message string, and an OK and Cancel ButtonViewModel (each having a label string and OnClick action)
* [SliderView](/Assets/Scripts/_ShinyUnity/UI/Views/SliderView.cs)
  * Binds to a float value to drive the UGUI slider component