---
published: true
layout: post
comments: true
---







## Space Engineering - Programming block

Since KeenSWH released the Programming block its now possible to build pretty good automations for your ships
and go as far as building fully automated ships. However, I notice that there is a distinct lack of interface design in the exposed API.

For now I'm going to dump a few bits of code that i'm working on here

### Speed calculator
Shamelessly adapted from a [Gist](https://gist.github.com/awstanley/eb72ef4aa8683b7f0cf3) by [A.W. Stanley](https://gist.github.com/awstanley)
{% gist YarekTyshchenko/8d5760966fc1b5d56c20 file-cruisecontrol-cs %}

How to use:

{% highlight c# %}
// Instantiate a speed calculator using the programming block for measurments
// You can use any block, something near the center of the ship so the rotation
// doesn't contribute to the overall speed.
SpeedCalculator sc = new SpeedCalculator((IMyTerminalBlock)Me);

// Storage malarky: Required because we need to store previous 3D Vector and time
// to calculate distance traveled and speed
SpeedCalculator.Store store = SpeedCalculator.Store.fromString(Storage);
sc.calculate(ref store);
Storage = store.ToString();

// Speed in Meters Per Second
double speed = sc.getSpeed();
// Seconds passed since last calculation
double delta = sc.getDeltaSeconds();
{% endhighlight %}

I'm still working on the store implementation, but this first iteration should flexible about the persistence method. The ser-de logic is hidden in the struct.

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
// Second parameter is the name of the panel to search for
LCD lcd = new LCD(GridTerminalSystem, "LCD Panel").clear();
lcd.writeLine("Speed: "+10);
lcd.writeLine("Seconds Delta: "+2);
{% endhighlight %}

An object should be created for each Panel. The interface is "fluid" so you can chain functions together `lcd.clear().writeLine("foo");` or `lcd.writeLine("foo").writeLine("bar");`, so you don't have to add newlines manually

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
            IMyTerminalBlock t = (IMyTerminalBlock)this.thrusters[i];
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
IMyThrust t = GridTerminalSystem.GetBlockWithName("CC (Forward) ") as IMyThrust;

// Add them to the cruise control system
cc.addThruster(t);

// Recompute on every update
cc.recompute(sc.getSpeed());
{% endhighlight %}

The thruster passed in need to be already overridden for this to work.
The naive implementation will just toggle the overridden thruster on and off dependent on the speed passed in.

### Computer loop
Its horrible that method dispatch isn't really possible with the restrictions in the programming block. Can't use reflection class.
It would be nice to somehow subclass the `Computer` put your own implementation there, possibly having a behaviour structure.
It is possible!:
{% highlight c# %}
void Main(string arg) {
	Action messageTarget = delegate() {
		Echo("Work within delegate");
	};
	messageTarget();
}
{% endhighlight %}

This code here is very experimental, and doesn't really give any advantage to just putting code straight into the main method. The goal of this is to both separate init stage from runtime and to offer a mechanism for timer speed management. Another neccessity is to provide a state machine responding to stimuli from outside. None of this is possible really without wrappers for the loop and the storage mechanisms.
{% highlight c# %}
public class Computer {
    private bool runOnce = false;

    // Returns false once
    public bool init() {
        bool val = this.runOnce;
        this.runOnce = true;
        return val;
    }

    private IMyTimerBlock timerBlock;

    public void setTimer(IMyTimerBlock timerBlock) {
        this.timerBlock = timerBlock;
    }

    public void triggerNow() {
        this.timerBlock.GetActionWithName("TriggerNow").Apply(this.timerBlock);
    }

    public void loop() {

    }
}
{% endhighlight %}

And to use create it before the main method:
{% highlight c# %}
Computer computer = new Computer();
SpeedCalculator sc;
LCD lcd;
void Main(string arg)  
{
    if (! computer.init()) {
        IMyTimerBlock timerBlock = GridTerminalSystem.GetBlockWithName("Timer Block") as IMyTimerBlock;
        computer.setTimer(timerBlock);

        sc = new SpeedCalculator((IMyTerminalBlock)Me);
        lcd = new LCD(GridTerminalSystem, "LCD");
    }

    //computer.loop();

    SpeedCalculator.Store store = SpeedCalculator.Store.fromString(Storage); 
    sc.calculate(ref store); 
    Storage = store.ToString(); 
 
    lcd.clear();
    lcd.writeLine("Speed: "+Math.Round(sc.getSpeed(), 3)); 
    lcd.writeLine("Seconds Delta: "+Math.Round(sc.getDeltaSeconds(), 3)); 
}
{% endhighlight %}

### Navigator
Navigational Computer is responsible for making maneuvers, changing the orientation of the ship, rather than translating ship's position.
{% highlight c# %}
public class Navigator {
    private Dictionary<ushort, IMyGyro> gyros = new Dictionary<ushort, IMyGyro>();
    public class Direction {
        public const ushort Left = 0;
        public const ushort Right = 1;
        public const ushort Up = 2;
        public const ushort Down = 3;
    }

    public void addGyro(ushort direction, IMyGyro gyro) {
         this.gyros.Add(direction, gyro);
    }

    public struct Maneuver {
        public bool isSet;
        public DateTime timeStarted;
        public ushort direction;
        public double startAngle;
        public double duration;
        public bool finished() {
            DateTime finishTime = this.timeStarted.AddSeconds(this.duration);
            return System.DateTime.Now >= finishTime;
        }

        public bool IsSet() {
            return this.isSet;
        }

        public string ToString() {
            DateTime finishTime = this.timeStarted.AddSeconds(this.duration);

            double done = (finishTime - System.DateTime.Now).TotalSeconds;
            return "D:" + this.direction + ", " + this.timeStarted.ToString() + " \nfor " + this.duration + " Done in "+ Math.Round(done, 3);
        }
    }

    private Maneuver activeManeuver;

    private void TurnOnGyro(ushort direction) {
        IMyTerminalBlock tb = (IMyTerminalBlock)this.gyros[direction];
        tb.GetActionWithName("OnOff_On").Apply(tb);
    }

    private void TurnOffGyro(ushort direction) {
        IMyTerminalBlock tb = (IMyTerminalBlock)this.gyros[direction];
        tb.GetActionWithName("OnOff_Off").Apply(tb);
    }

    public Maneuver Turn(double seconds, ushort direction) {
        Maneuver m = new Maneuver();
        m.isSet = true;
        m.timeStarted = System.DateTime.Now;
        m.direction = direction;
        m.duration = seconds;
        stop();
        TurnOnGyro(direction);
        return m;
    }

    public Maneuver MakeRandomTurn() {
        // Pick a direction
        Random rnd = new Random();
        ushort direction = (ushort)rnd.Next(0, 4);
        // Pick an angle (duration)
        double duration = rnd.NextDouble() * 3;
        return Turn(duration, direction);
    }

    public void stop() {
        TurnOffGyro(Direction.Left);
        TurnOffGyro(Direction.Right);
        TurnOffGyro(Direction.Up);
        TurnOffGyro(Direction.Down);
    }
}
{% endhighlight %}

Usage:
{% highlight c# %}
Navigator.Maneuver m;
void Main(string arg) {
    Navigator nav = new Navigator();
    nav.addGyro(Navigator.Direction.Left, GridTerminalSystem.GetBlockWithName("Gyroscope Left") as IMyGyro);
    nav.addGyro(Navigator.Direction.Right, GridTerminalSystem.GetBlockWithName("Gyroscope Right") as IMyGyro);
    nav.addGyro(Navigator.Direction.Up, GridTerminalSystem.GetBlockWithName("Gyroscope Up") as IMyGyro);
    nav.addGyro(Navigator.Direction.Down, GridTerminalSystem.GetBlockWithName("Gyroscope Down") as IMyGyro);

    if (arg == "left") {
        m = nav.Turn(5, Navigator.Direction.Left);
    }
    
    // If the maneuver is done
    if (m.finished()) {
        nav.stop();
        m.isSet = false;
    }
}
{% endhighlight %}

The interface here is somewhat badly designed as I would like to make the navigator work with angles rather than seconds, but as it is it should allow decent automatic control of the ship
