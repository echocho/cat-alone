# A Review of Design Patterns
[TODO] description. what this is about.
Overview of all 23 design patterns and category

## Strategy Pattern
### Definition  
Strategy is a behavioral pattern that encapsulates a family of algorithms and enables selecting a specific algorithm at runtime.
Strategy lets the algorithm vary independently of clients that use it.
It's useful when you want the application to do X, but X is not clear yet and would change in the near future.

### Example
Let's say we are building a video game. When defining characters' movement, we want them to run or walk. How to implement it? One way is through inheritance:
![game-character-inheritance-class-diagram.png](pics/game-character-inheritance-class-diagram.png)
Our implementation may look something like this: 
```Java
public abstract class Character {

    public abstract String getName();

    public void walk() {
        System.out.println("walking..");
    }

    public void run() {
        System.out.println("running!");
    }

}

public class Queen extends Character {

    @Override
    public String getName() {
        return "Elegant Queen";
    }

}

public class King extends Character {

    @Override
    public String getName() {
        return "Ugly King";
    }
}

```
So far, everything works. However, what if we want to add new characters in the future, and they have fancier way to move? For example, 
- Dr.Who, who teleports
- Mermaid, who swims
- Rabbit, who burrows underground.

How to implement these requirements? With the current implementation, we'll need to add `swim()`, `teleport()` and `burrowUnderground()` in the parent class `Character`. 
```java
public abstract class Character {

    public abstract String getName();

    public void walk() {
        System.out.println("Walking..");
    }

    public void run() {
        System.out.println("Running!");
    }

    public void teleport() {
        System.out.println("Fancy teleport!");
    }

    public void swim() {
        System.out.println("Swimming.");
    }

    public void burrowUnderground() {
        System.out.println("Burrowing underground - you can't see me!");
    }
}
```
However, by doing that we give King, Queen and Soldier all these new capabilities that they shouldn't have!
```java
public class CharacterSimulator {

    public static void main(String[] args) {
        Character queen = new Queen();
        System.out.println(queen.getName());
        queen.walk();
        queen.run();
        queen.burrowUnderground();
        queen.teleport();
    }
}
```
Output:
```
Elegant Queen
Walking..
Running!
Burrowing underground - you can't see me!  // not what we want Queen to do
Fancy teleport!                            // not what we want Queen to do
```


We could override these new methods in `Queen`, `King` and `Soldier` and return early. However, we need to keep doing this whenever we add new characters and new movement in the future. That's no convenience.
```java
public class Queen extends Character {

    @Override
    public String getName() {
        return "Elegant Queen";
    }

    @Override
    public void burrowUnderground() {
        System.out.println("Queen doesn't burrow underground!");
    }

}
```
To make our lives easier, we need to find a way to change movement without changing our existing characters - that's when Strategy pattern comes in handy. 
`Character` has two types of method, one is common for all characters and won't change, like `getName()`; the other is changeable like movements.
We can encapsulate movements in an interface and let each character uses a different movement at runtime.

Following is the new class diagram using Strategy pattern. 
![game-character-strategy-pattern-class-diagram.png](pics/game-character-strategy-pattern-class-diagram.png)

We need to do the following in our code:
1. Create an interface called `MovementBehavior`.
2. Create different types of movement that implement the interface, i.e. `Teleport`, `BurrowUnderground`, `Swim`.
3. Extract `walk()` and `run()` from `Character` and make them implementations of `MovementBehavior`, i.e. `Run`, `Walk`. 
4. Use specific movement in specific character.

```java
// Character related 
public abstract class Character {

    MovementBehavior movementBehavior;

    public abstract String getName();

    public void move() {
        movementBehavior.move();
    }

}

public class DrWho extends Character {

    @Override
    public String getName() {
        return "Cool Dr.Who";
    }

    public DrWho() {
        movementBehavior = new Teleport();
    }

}

public class Queen extends Character {

    @Override
    public String getName() {
        return "Elegant Queen";
    }

    public Queen() {
        movementBehavior = new Walk();
    }
}
// ... similar changes to `King` and `Soldier`

// Movement related
public interface MovementBehavior {
    void move();
}

public class Teleport implements MovementBehavior {

    @Override
    public void move() {
        System.out.println("Check this out - I'm teleporting!");
    }
}

public class BurrowUnderground implements MovementBehavior {

    @Override
    public void move() {
        System.out.println("Burrowing underground - you can't see me!");
    }
}

public class Walk implements MovementBehavior {

    @Override
    public void move() {
        System.out.println("Walking..");
    }
}
// ... similar changes to `Swim` and `Run`
```
Now, let's test our code:
```java
public class CharacterSimulator {

    public static void main(String[] args) {
        Character drWho = new DrWho();
        drWho.move();

        Character queen = new Queen();
        queen.move();
    }
}
```
```
// output:
Check this out - I'm teleporting!
Walking..
```
So far, we isolate movement from `Character` so that whenever there are new types of movement added, we don't need to change existing characters.
    But what if we want characters to have new moves? To do that, we can simply add a setter `setMovementBehavior(MovementBehavior mb)` in `Character`:
```java
public abstract class Character {

    MovementBehavior movementBehavior;

    public abstract String getName();

    public void move() {
        movementBehavior.move();
    }

    public void setMovementBehavior(MovementBehavior mb) {
        this.movementBehavior = mb;
    }

}
```
Test and see if Dr.Who can move differently now:
```java
public class CharacterSimulator {

    public static void main(String[] args) {
        Character drWho = new DrWho();
        drWho.move();

        drWho.setMovementBehavior(new BurrowUnderground());
        drWho.move();
    }
}
```

```
// Output
Check this out - I'm teleporting!
Burrowing underground - you can't see me!
```
Well, yes, he can!

### Summary
From the above example, we can see it's important to **identify areas in your application that may need to change and isolate them from other areas that do not change**. 
In cases where properties or methods are prone to change or unclear during development, we should **use composition more, inheritance less**. And as always, **programing to the interface, not the implementation** would make our code more flexible.


## Observer Pattern
### Definition  
In the observer pattern, one object (subject) maintains a list of objects (observers) that are interested in its state changes. Whenever there's any changes, the subject notify its observers, usually by calling one of their methods.
The observer pattern is often implemented synchronously and within an application. 

### Example
Weather forecasting is a classic example. We have an object `WeatherData` that contains the weather measurements collection from a weather station. We want to build some weather dashboards and want to refresh the dashboards when there's any changes data measurements. In addition, we want to be flexible in creating new dashboard, updating existing dashboards, or deleting old dashboards.
To do this, we create 3 interfaces: Subject, Observer, and DisplayElement. Subject handles observer registrations and notification. Observer provides a method for Subject to call to get notification. DisplayElement decides how data are displayed.
Class diagram and code implementation can be found online easily. Following is the code from Head First Design Patterns, with some improvements.
```java
public interface Subject {

    void registerObserver(Observer o);

    void unregisterObserver(Observer o);

    void notifyObservers();
}

public class WeatherData implements Subject {

    private float temperature;
    private float humidity;
    private float pressure;
    private final List<Observer> observers = new ArrayList<>();


    public void measurementsChanged() {
        notifyObservers();

    }

    @Override
    public void registerObserver(Observer o) {
        int idx = observers.indexOf(o);
        if (idx == -1) {
            observers.add(o);
        }
    }

    @Override
    public void unregisterObserver(Observer o) {
        int idx = observers.indexOf(o);
        if (idx >= 0) {
            observers.remove(idx);
        }
    }

    @Override
    public void notifyObservers() {
        WeatherMeasurement measurement = new WeatherMeasurement.Builder()
                .withTemperature(temperature)
                .withHumidity(humidity)
                .withPressure(pressure).build();
        for (Observer o : observers) {
            try { // put update in try block so that exception occurs in previous observer won't impact the next
                o.update(measurement);
            } catch (Exception e) {
                // log the error or retry
            }

        }
    }

    // method for test
    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}

public class WeatherMeasurement {
    private final float temperature;
    private final float humidity;
    private final float pressure;

    public float getTemperature() {
        return temperature;
    }

    public float getHumidity() {
        return humidity;
    }

    public float getPressure() {
        return pressure;
    }

    private WeatherMeasurement(Builder builder) {
        this.temperature = builder.temperature;
        this.humidity = builder.humidity;
        this.pressure = builder.pressure;
    }

    // Use Builder Pattern here because there may be more measurements in the future
    public static class Builder {
        private float temperature;
        private float humidity;
        private float pressure;

        public Builder() {

        }

        public Builder withTemperature(float temperature) {
            this.temperature = temperature;
            return this;
        }

        public Builder withHumidity(float humidity) {
            this.humidity = humidity;
            return this;
        }

        public Builder withPressure(float pressure) {
            this.pressure = pressure;
            return this;
        }

        public WeatherMeasurement build() {
            return new WeatherMeasurement(this);
        }
    }
}

public interface Observer {

    // Encapsulate all weather data in WeatherMeasurement, instead of using specific measurement like temperature, humidity, pressure in method signature.
    // In this way, we decouple Observer from future changes of weather data model.
    void update(WeatherMeasurement weatherMeasurement);
}

public class CurrentConditionsDisplay implements Observer, DisplayElement {

    private Subject weatherData;
    private float temperature;
    private float humidity;
    private float pressure;

    public CurrentConditionsDisplay(Subject weatherData) {
        this.weatherData = weatherData;
        weatherData.registerObserver(this);
    }

    @Override
    public void display() {
        System.out.println("Current conditions: " + temperature + "F degrees and " + humidity + "% humidity and " + pressure + " Pa pressure");
    }

    @Override
    public void update(WeatherMeasurement weatherMeasurement) {
        this.temperature = weatherMeasurement.getTemperature();
        this.humidity = weatherMeasurement.getHumidity();
        this.pressure = weatherMeasurement.getPressure();
        display();
    }
}

// Test class
public class DashboardSimulator {

    public static void main(String[] args) {
        WeatherData weatherData = new WeatherData();
        CurrentConditionsDisplay currentConditionsDisplay = new CurrentConditionsDisplay(weatherData);
        StatisticsDisplay statisticsDisplay = new StatisticsDisplay(weatherData);
        weatherData.setMeasurements(11.11F, 22.22F, 33.33F);
        weatherData.setMeasurements(34.34F, 88.88F, 109.109F);

        weatherData.unregisterObserver(currentConditionsDisplay);
        weatherData.setMeasurements(10.10F, 77.77F, 23.23F);

    }
}
```
```
// Output
Current conditions: 11.11F degrees and 22.22% humidity and 33.33 Pa pressure
Temperature high: 11.11 degrees, temperature low: 11.11 degrees; humidity high: 22.22, humidity low: 22.22
Current conditions: 34.34F degrees and 88.88% humidity and 109.109 Pa pressure
Temperature high: 34.34 degrees, temperature low: 11.11 degrees; humidity high: 88.88, humidity low: 22.22
Temperature high: 34.34 degrees, temperature low: 10.1 degrees; humidity high: 88.88, humidity low: 22.22
```

### Things to note when using observer pattern
#### 1. Handle exception carefully when notifying observers
Observer pattern is often implemented synchronously, which means if you have a list of observers, you'll call `observer.update()` synchronously. If there's error in the previous observer, and it's uncaught, the next observer won't be notified.

#### 2. For tasks that need long time to process and real-time processing is not necessary, consider using publisher-subscriber pattern.
The publisher-subscriber pattern is for asynchronous communication, with the help of a third component, usually a message broker or an event bus.
One benefit of using publisher-subscriber pattern over observer pattern is that the former is not blocking.
Another benefit is that message brokers usually have retry mechanism built in; while you need to implement your own retry mechanism when using the observer pattern.


## Singleton Pattern
ways of implementing a singleton
potential problems of singleton and how to handle them (threadsafe, performance, etc)
explanation with code

## Decorator Pattern
java vs python




## Reference
- Head First Design Patterns, by Eric Freeman, Elisabeth Robson, Bert Bates, Kathy Sierra

- https://stackoverflow.com/questions/370258/real-world-example-of-the-strategy-pattern