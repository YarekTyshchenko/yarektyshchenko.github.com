---
published: true
layout: post
comments: true
---



## Space Engineering - Programming block

Since KeenSH released the Programming block its now possible to build pretty good automations for your ships
and go as far as building fully automated ships. However, I notice that there is a distinct lack of interface design in the exposed API.

For now I'm going to dump a few bits of code that i'm working on here

### Speed calculator

{% highlight c# %}
public class SpeedCalculator {
    public struct Store {
        // Previous Time
        public DateTime time;
        // Previous position
        public Vector3D position;

        public static Store fromString(string storage) {
            Store store = new Store();
            // Get the fragments (or get one fragment) 
            string[] Fragments = storage.Split('|');
            if(Fragments.Length == 2) {
                // Extract time
                store.time = System.DateTime.FromBinary(Convert.ToInt64(Fragments[0]));

                // Vomit a bit here because this is how we have to store variables at the moment ...
                string[] Coords = Fragments[1].Split(',');
                double X = Math.Round(Convert.ToDouble(Coords[0]), 4); 
                double Y = Math.Round(Convert.ToDouble(Coords[1]), 4); 
                double Z = Math.Round(Convert.ToDouble(Coords[2]), 4); 
                store.position = new Vector3D(X, Y, Z);
            } else {            
                store.time = System.DateTime.Now;
                store.position = new Vector3D(0);
            }
            return store;
        }

        public string ToString() {
            // Get coordinates (VRageMath.Vector3D, so pull it in the ugly way) 
            double x = Math.Round(this.position.GetDim(0), 4); 
            double y = Math.Round(this.position.GetDim(1), 4); 
            double z = Math.Round(this.position.GetDim(2), 4);

            // Time|X,Y,Z
            string newStorage = this.time.ToBinary().ToString() + "|" + 
                x.ToString() + "," + y.ToString() + "," + z.ToString();
            return newStorage;
        }
    }

    IMyTerminalBlock origin;
    double delta = 0.0;
    double speed = 0.0;

    public SpeedCalculator(IMyTerminalBlock origin) {
        this.origin = origin;
    }

    public double getDeltaSeconds() {
        return this.delta;
    }

    public double getSpeed() {
        return this.speed;
    }

    public void calculate(ref Store store) {
        // Get times 
        DateTime TimeNow = System.DateTime.Now;
        DateTime OldTime = store.time;

        // We have 's' for m/s. 
        this.delta = (TimeNow - OldTime).TotalSeconds;
        
        // Caluculate distance
        Vector3D currentVector = this.origin.GetPosition();
        double Distance = Vector3D.Distance(currentVector, store.position);

        // If the base coordinates 
        if (Distance != 0 && this.delta != 0)
        {
            // We have our distance 
            double Speed = Distance / this.delta;
            this.speed = Speed;
        } else {
            this.speed = 0.0;
        }

        // Update the store with current coordinates
        store.time = TimeNow;
        store.position = currentVector;
    }
}

{% endhighlight %}

How to use:

{% highlight c# %}
SpeedCalculator sc = new SpeedCalculator((IMyTerminalBlock)Me);
SpeedCalculator.Store store = SpeedCalculator.Store.fromString(Storage);
sc.calculate(ref store);
Storage = store.ToString();

// Speed in Meters Per Second
double speed = sc.getSpeed();
// Seconds passed since last calculation
double delta = sc.getDeltaSeconds();
{% endhighlight %}

I'm still working on the store implementation, but this first iteration should allow you to use whatever persistance method you want.

### LCD Panel
{% highlight c# %}
class LCD {
    IMyTextPanel lcd;
    
    // Grid terminal system is needed here so the search happens in the right grid
    public LCD(IMyGridTerminalSystem gts, string panelName) {
        this.lcd = gts.GetBlockWithName(panelName) as IMyTextPanel;
        this.lcd.ShowTextureOnScreen();
        this.lcd.ShowPublicTextOnScreen();
    }

    public LCD writeText(string text) {
        this.lcd.WritePublicText(text, true);
        return this;
    }

    public LCD writeLine(string line) {
        this.writeText(line+"\n");
        return this;
    }

    public LCD clear() {
        this.lcd.WritePublicText("", false);
        return this;
    }
}
{% endhighlight %}

How to use:
{% highlight c# %}
LCD lcd = new LCD(GridTerminalSystem, "LCD Panel").clear();
lcd.writeLine("Speed: "+10);
lcd.writeLine("Seconds Delta: "+2);
{% endhighlight %}

An object is created for each Panel. The interface is "fluid" so you can chain functions together `lcd.clear().writeLine("foo");` or `lcd.writeLine("foo").writeLine("bar");`, so you don't have to add newlines manually

### Cruise Control
{% highlight c# %}
public class CruiseControl {
    private double targetSpeed = 0.0;
    private List<IMyThrust> thrusters = new List<IMyThrust>();

    public void setTarget(double targetSpeed) {
        this.targetSpeed = targetSpeed;
    }
    public void addThruster(IMyThrust thruster) {
        this.thrusters.Add(thruster);
    }
    public void recompute(double currentSpeed) {
        // recompute the thruster override settings based on the current speed
        this.simpleImplementation(currentSpeed);
    }

    private void simpleImplementation(double currentSpeed) {
        for(var i = 0; i < this.thrusters.Count; i++) {
            IMyTerminalBlock t = this.thrusters[i];
            // If going too fast
            if (currentSpeed > targetSpeed) {
                // Turn all thrusters off
                t.GetActionWithName("OnOff_Off").Apply(t);
            } else {
                // Turn all thrusters on
                t.GetActionWithName("OnOff_On").Apply(t);
            }
        }
    }
}
{% endhighlight %}

How to use it:
{% highlight c# %}
// Create an instance
CruiseControl cc = new CruiseControl(); 
// Set its target speed
cc.setTarget(23.5);

// Find all the thrusters that point in the direction of where you are going
IMyThrust t = GridTerminalSystem.GetBlockWithName("CC (Forward)") as IMyThrust;

// Add them to the cruise control system
cc.addThruster(t);

// Recompute on every update
cc.recompute(sc.getSpeed());
{% endhighlight %}

The thruster passed in need to be already overridden for this to work.
The naive implementation will just toggle the overridden thruster on and off dependent on the speed passed in.