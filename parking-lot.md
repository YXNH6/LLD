## 1. Entities

### 1.1 ParkingLot

**Responsibility:** Entry point for vehicle entry/exit, delegates work to floors and transactions.

**Attributes**

* `List<ParkingFloor> floors`

**Methods**

* `ParkingSpot allocateSpot(Vehicle vehicle)`
* `void releaseSpot(ParkingSpot spot)`

---

### 1.2 ParkingFloor

**Responsibility:** Holds parking spots for a single floor.

**Attributes**

* `int floorNumber`
* `List<ParkingSpot> parkingSpots`

**Methods**

* `ParkingSpot getAvailableSpot(VehicleSize size)`

---

### 1.3 ParkingSpot

**Attributes**

* `int id`
* `int floor`
* `int spotNumber`
* `VehicleSize size`
* `boolean isOccupied`

**Methods**

* `void occupy()`
* `void free()`

---

### 1.4 Vehicle

**Attributes**

* `int id`
* `String licensePlate`
* `VehicleSize size`

**Methods**

* Constructor and getters and setters

---

### 1.5 ParkingTransaction

**Attributes**

* `int id`
* `Vehicle vehicle`
* `ParkingSpot parkingSpot`
* `LocalDateTime entryTime`
* `LocalDateTime exitTime`
* `double fee`

**Methods**

* `void closeTransaction(LocalDateTime exitTime, double fee)`

---

### 1.6 Enums

enum VehicleSize {
    MOTORCYCLE, CAR, BUS
}


---

## 2. Spot Allocation Algorithm

```java
public ParkingSpot allocateSpot(Vehicle vehicle) {
    for (ParkingFloor floor : floors) {
        ParkingSpot spot = floor.getAvailableSpot(vehicle.getSize());
        if (spot != null) {
            spot.occupy();
            return spot;
        }
    }
    return null;
}
```

---

## 3. Fee Calculation Logic 

```java
public class FeeCalculator {

    public static double calculateFee(VehicleSize size, long durationHours) {
        switch (size) {
            case MOTORCYCLE:
                return durationHours * 100;
            case CAR:
                return durationHours * 200;
            case BUS:
                return durationHours * 300;
            default:
                return 0;
        }
    }
}
```

---
## 4. Core Service

### ParkingService

```java
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.*;

public class ParkingService {

    private ParkingLot parkingLot;
    private Map<Integer, ParkingTransaction> transactions = new HashMap<>();
    private int transactionCounter = 1;

    public ParkingService(ParkingLot parkingLot) {
        this.parkingLot = parkingLot;
    }

    public synchronized int handleVehicleEntry(Vehicle vehicle) {
        ParkingSpot spot = parkingLot.allocateSpot(vehicle);

        if (spot == null) {
            throw new RuntimeException("No available parking spots for vehicle size");
        }

        ParkingTransaction transaction = new ParkingTransaction(
                transactionCounter,
                vehicle.getId(),
                spot.getId(),
                LocalDateTime.now()
        );

        transactions.put(transactionCounter, transaction);
        return transactionCounter++;
    }

    public synchronized double handleVehicleExit(int transactionId, VehicleSize vehicleSize) {
        ParkingTransaction transaction = transactions.get(transactionId);

        if (transaction == null || transaction.getExitTime() != null) {
            throw new RuntimeException("Invalid transaction");
        }

        LocalDateTime exitTime = LocalDateTime.now();
        long hours = Duration.between(transaction.getEntryTime(), exitTime).toHours();
        if (hours == 0) hours = 1;

        double fee = FeeCalculator.calculateFee(vehicleSize, hours);

        transaction.closeTransaction(exitTime, fee);

        parkingLot.releaseSpotById(transaction.getParkingSpotId());

        return fee;
    }
}
```

---

## 5. Sample Code

### ParkingFloor

```java
public class ParkingFloor {
    private int floorNumber;
    private List<ParkingSpot> parkingSpots;

    public ParkingFloor(int floorNumber, List<ParkingSpot> parkingSpots) {
        this.floorNumber = floorNumber;
        this.parkingSpots = parkingSpots;
    }

    public ParkingSpot getAvailableSpot(VehicleSize size) {
        for (ParkingSpot spot : parkingSpots) {
            if (!spot.isOccupied() && spot.getSize() == size) {
                return spot;
            }
        }
        return null;
    }
}
```

### ParkingSpot

```java
public class ParkingSpot {
    private int id;
    private int floor;
    private int spotNumber;
    private VehicleSize size;
    private boolean isOccupied;

    public void occupy() {
        isOccupied = true;
    }

    public void free() {
        isOccupied = false;
    }

    // getters
}
```

### ParkingTransaction

```java
import java.time.LocalDateTime;

public class ParkingTransaction {
    private int id;
    private int vehicleId;
    private int parkingSpotId;
    private LocalDateTime entryTime;
    private LocalDateTime exitTime;
    private double fee;

    public ParkingTransaction(int id, int vehicleId, int parkingSpotId, LocalDateTime entryTime) {
        this.id = id;
        this.vehicleId = vehicleId;
        this.parkingSpotId = parkingSpotId;
        this.entryTime = entryTime;
    }

    public void closeTransaction(LocalDateTime exitTime, double fee) {
        this.exitTime = exitTime;
        this.fee = fee;
    }

    // getters
}
```

---

## 6. Driver Code 

```java
public class Main {
    public static void main(String[] args) {

        Vehicle vehicle = new Vehicle(1, "KA-01-1234", VehicleSize.CAR);

        ParkingService parkingService = new ParkingService(parkingLot);

        int transactionId = parkingService.handleVehicleEntry(vehicle);
        System.out.println("Vehicle entered. Transaction ID: " + transactionId);

        double fee = parkingService.handleVehicleExit(transactionId, vehicle.getSize());
        System.out.println("Parking fee: ₹" + fee);
    }
}
```
