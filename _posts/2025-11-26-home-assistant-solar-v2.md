---
title: "Home Assistant solar energy management V2"
date: 2025-11-26
categories: [DIY]
tags: [Homelab, Python, Software Engineering]
header:
    overlay_image: /assets/images/2025-11-26-home-assistant-solar-v2/overlay.png
    caption: ""
---

Recently I decided to learn Python seriously and studied Fluent Python: Clear, Concise, and Effective Programming by Luciano Ramalho.
The hardest part was finding a project to apply the new skills.
At work I mostly use Python to write small scripts and Apache Airflow DAGs.
I wanted something more challenging with complex domain logic and real-world data ⚙️📊

![Fluent Python](/assets/images/2025-11-26-home-assistant-solar-v2/fluent_python_book_cover.jpg)

## Introduction

This spring I installed a photovoltaic (PV) system on my roof to generate renewable energy for my home. Soon after, I wrote a Home Assistant automation to optimize solar use. You can read more in my previous post [Home Assistant solar energy management](/blog/2025/04/12/home-assistant-solar/).

Summer, with plenty of sunshine, didn't challenge that YAML-based setup. As autumn and winter approached with shorter days, less predictable weather, and heating energy consumption, I realized I needed something more robust than a collection of YAML automations.

{: .notice--info}
I found an excellent opportunity to apply lessons from Fluent Python 😂

After two to three months of tinkering in my spare time, I implemented a new solar and heating energy management system using [AppDaemon](https://github.com/AppDaemon/appdaemon), a sandboxed execution environment for Home Assistant. Here are some stats about the project:

- 📦 2,529 lines of production code across 55 Python files
- 🧪 4,248 lines of test code in 37 test files
- 🧾 476 test scenarios executed (170 test functions expanded through parametrization)
- ✅ 94% test coverage

I originally planned to describe every part of the project, but that would be too much for most readers. So I split the post into three parts:

1. Configuration snippets that reveal the system's complexity.
2. A closer look at a few Python techniques I used.
3. Selected algorithms and how they work in practice.

## AppDaemon Solar and HVAC applications configuration

The first snippet covers configuration needed for implementing solar energy management logic:

```python
configuration = SolarConfiguration(
    # nominal battery capacity
    battery_capacity=EnergyKwh(10.0),
    # nominal battery voltage
    battery_voltage=BatteryVoltage(52.0),
    # maximum battery discharge/charge current
    battery_maximum_current=BatteryCurrent(80.0),
    # minimum reserve SOC
    battery_reserve_soc_min=BatterySoc(20.0),
    # margin above minimum reserve SOC
    battery_reserve_soc_margin=BatterySoc(8.0),
    # upper limit when charging from the grid
    battery_reserve_soc_max=BatterySoc(90.0),
    # indoor temperature setpoint to estimate heating needs
    temp_in=Celsius(21.0),
    # outdoor temperature threshold to apply heating energy consumption in eco mode
    temp_out_threshold=Celsius(2.0),
    # coefficient of heat-pump performance at 7 degrees Celsius
    heating_cop_at_7c=4.0,
    # coefficient representing building heat loss rate in kW/°C
    heating_h=0.18,
    # outdoor temperature if weather forecast isn't available
    temp_out_fallback=Celsius(2.0),
    # outdoor humidity if weather forecast isn't available
    humidity_out_fallback=80.0,
    # regular consumption when in away mode
    regular_consumption_away=EnergyKwh(0.35),
    # consumption during daytime
    regular_consumption_day=EnergyKwh(0.5),
    # consumption during evening
    regular_consumption_evening=EnergyKwh(0.8),
    # threshold for exporting PV energy, net price
    pv_export_min_price_margin=EnergyPrice.pln_per_mwh(Decimal(200)),
    # threshold for exporting battery energy, net price
    battery_export_threshold_price=EnergyPrice.pln_per_mwh(Decimal(1000)),
    # skip battery export below this threshold
    battery_export_threshold_energy=EnergyKwh(1.0),
    # start time of night low tariff period (with margin)
    night_low_tariff_time_start=time.fromisoformat("22:05:00"),
    # end time of night low tariff period (with margin)
    night_low_tariff_time_end=time.fromisoformat("06:55:00"),
    # start time of day low tariff period (with margin)
    day_low_tariff_time_start=time.fromisoformat("13:05:00"),
    # end time of day low tariff period (with margin)
    day_low_tariff_time_end=time.fromisoformat("15:55:00"),
)
```

Static configuration is complemented by dynamic state information about the system taken from Home Assistant sensors and integrations:

```python
class SolarState:
    battery_soc: BatterySoc # current battery state of charge
    battery_reserve_soc: BatterySoc # current battery reserve state of charge
    is_away_mode: bool # away mode status
    is_eco_mode: bool # eco mode status
    inverter_storage_mode: StorageMode # current inverter storage mode
    is_slot1_discharge_enabled: bool # whether slot 1 discharge is enabled
    slot1_discharge_time: str # discharge time for slot 1
    slot1_discharge_current: BatteryCurrent # discharge current for slot 1
    hvac_heating_mode: str # heating mode
    hourly_price: EnergyPrice # current hourly energy price
    pv_forecast_today: list # today's PV forecast
    pv_forecast_tomorrow: list # tomorrow's PV forecast
    weather_forecast: dict | None # weather forecast data
    price_forecast: list | None # energy price forecast
```

HVAC (Heating, Ventilation, and Air Conditioning) control is another complex part of the system. Here is a snippet of the configuration:

```python
configuration = HvacConfiguration(
    # domestic hot water temperature
    dhw_temp=Celsius(48.0),
    # domestic hot water temperature in eco mode
    dhw_temp_eco=Celsius(40.0),
    # when to start boosting DHW depends on temperature difference
    dhw_delta_temp=Celsius(6.0),
    # 5 minutes after low tariff starts to avoid clocks drift issues
    dhw_boost_start=time.fromisoformat("13:05:00"),
    # 5 minutes before high tariff starts to avoid clocks drift issues
    dhw_boost_end=time.fromisoformat("15:55:00"),
    # heating temperature
    heating_temp=Celsius(20.0),
    # heating temperature in eco mode
    heating_temp_eco=Celsius(18.0),
    # heating boost delta
    heating_boost_delta_temp=Celsius(1.0),
    # heating boost delta in eco mode
    heating_boost_delta_temp_eco=Celsius(2.0),
    # 5 minutes after low tariff starts to avoid clocks drift issues
    heating_boost_time_start_eco_on=time.fromisoformat("22:05:00"),
    # 15 minutes before high tariff starts because stop heating takes longer
    heating_boost_time_end_eco_on=time.fromisoformat("06:45:00"),
    # 1 hour before wake up time
    heating_boost_time_start_eco_off=time.fromisoformat("05:00:00"),
    # 1 hour before bed time
    heating_boost_time_end_eco_off=time.fromisoformat("21:00:00"),
    # cooling temperature
    cooling_temp=Celsius(24.0),
    # cooling temperature in eco mode
    cooling_temp_eco=Celsius(26.0),
    # cooling boost delta
    cooling_boost_delta_temp=Celsius(2.0),
    # cooling boost delta in eco mode
    cooling_boost_delta_temp_eco=Celsius(2.0),
    # cool when there is plenty of solar energy
    cooling_boost_time_start_eco_on=time.fromisoformat("12:00:00"),
    cooling_boost_time_end_eco_on=time.fromisoformat("16:00:00"),
    # extends cooling period a bit when eco mode is off
    cooling_boost_time_start_eco_off=time.fromisoformat("10:00:00"),
    cooling_boost_time_end_eco_off=time.fromisoformat("18:00:00"),
)
```

And here is a snippet of the dynamic HVAC state:

```python
class HvacState:
    is_eco_mode: bool  # eco mode status
    dhw_actual_temperature: Celsius  # actual domestic hot water temperature
    dhw_target_temperature: Celsius  # target domestic hot water temperature
    indoor_actual_temperature: Celsius  # actual indoor temperature
    heating_target_temperature: Celsius  # target heating temperature
    heating_mode: str  # heating mode
    cooling_target_temperature: Celsius  # target cooling temperature
    cooling_mode: str  # cooling mode
    temperature_adjustment: Celsius  # temperature adjustment, +-1 degree
```

If you'd like to see the complete implementation, explore the source on GitHub: <https://github.com/mkuthan/home-assistant-appdaemon>. The main branch contains the production code I run at home.

## Python goodies

Let's move to the second part of the post, where I describe some interesting Python techniques I applied in the project.

### Typing

As a seasoned Java and Scala developer, I appreciate strong typing.
In Python typing is optional, but I decided to use it extensively to check it's maturity.
I configured my build and CI pipeline to use [pyright](https://github.com/microsoft/pyright) from Microsoft and [ty](https://github.com/astral-sh/ty) from Astral for type checking.

"Pyright" has the advantage of being fully compatible with Pylance in VS Code. However, I found "ty" to be blazingly fast — like other Astral tools such as ruff. The Astral tool is the clear winner in this comparison.

```shell
$ time pyright
Found 95 source files
0 errors, 0 warnings, 0 informations

real    0m2.661s
user    0m3.672s
sys     0m0.259s
```

```shell
$ time ty check
INFO Indexed 95 file(s) in 0.003s
All checks passed!

real    0m0.125s
user    0m0.250s
sys     0m0.059s
```

### Value classes

Around 2013 I read Domain-Driven Design: Tackling Complexity in the Heart of Software by Eric Evans.
One of the key takeaways was to use value objects to represent domain concepts instead of bunch of primitive types, like float for energy, temperature, etc.
I implemented value classes in Python using the `@dataclass(frozen=True)` decorator and a set of dunder (magic) methods for seamless integration with the Python SDK. For example:

```python
@dataclass(frozen=True, order=True)
class EnergyKwh:
    _ZERO_VALUE: ClassVar[float] = 0.0

    value: float

    def __add__(self, other: "EnergyKwh") -> "EnergyKwh":
        return EnergyKwh(value=self.value + other.value)

    def __sub__(self, other: "EnergyKwh") -> "EnergyKwh":
        return EnergyKwh(value=self.value - other.value)

    def __truediv__(self, other: "EnergyKwh") -> float:
        if other == ENERGY_KWH_ZERO:
            raise ValueError("Cannot divide by zero energy")
        return self.value / other.value

    def __neg__(self) -> "EnergyKwh":
        return EnergyKwh(value=-self.value)

    def __str__(self) -> str:
        return f"{self.value:.2f}kWh"


ENERGY_KWH_ZERO = EnergyKwh(EnergyKwh._ZERO_VALUE)
```

```python
class Celsius:
    _ZERO_VALUE: ClassVar[float] = 0.0

    value: float

    def __add__(self, other: "Celsius") -> "Celsius":
        return Celsius(value=self.value + other.value)

    def __sub__(self, other: "Celsius") -> "Celsius":
        return Celsius(value=self.value - other.value)

    def __mul__(self, other: float) -> "Celsius":
        return Celsius(value=self.value * other)

    def __truediv__(self, other: "Celsius") -> float:
        if other == CELSIUS_ZERO:
            raise ValueError("Cannot divide by zero temperature")
        return self.value / other.value

    def __round__(self) -> "Celsius":
        return Celsius(value=floor(self.value + 0.5))

    def __str__(self) -> str:
        return f"{self.value:.1f}°C"


CELSIUS_ZERO = Celsius(Celsius._ZERO_VALUE)
```

```python
@dataclass(frozen=True, order=True)
class BatterySoc:
    _MIN_VALUE: ClassVar[float] = 0.0
    _MAX_VALUE: ClassVar[float] = 100.0

    value: float

    def __post_init__(self) -> None:
        if not self._MIN_VALUE <= self.value <= self._MAX_VALUE:
            raise ValueError(f"Battery SOC must be between {self._MIN_VALUE} and {self._MAX_VALUE}, got {self.value}")

    def __add__(self, other: "BatterySoc") -> "BatterySoc":
        return BatterySoc(value=min(self.value + other.value, self._MAX_VALUE))

    def __sub__(self, other: "BatterySoc") -> "BatterySoc":
        return BatterySoc(value=max(self.value - other.value, self._MIN_VALUE))

    def __round__(self) -> "BatterySoc":
        return BatterySoc(value=floor(self.value + 0.5))

    def __str__(self) -> str:
        return f"{self.value:.2f}%"


BATTERY_SOC_MIN = BatterySoc(BatterySoc._MIN_VALUE)
BATTERY_SOC_MAX = BatterySoc(BatterySoc._MAX_VALUE)
```

With value classes I gained several benefits:

- With dataclass you get boilerplate code generation for free (`__init__`, `__eq__`, etc.)
- With frozen dataclass you get immutability for free
- With order=True you get comparison operators for free
- The type checker warns if you try to add an EnergyKwh value to a Celsius temperature
- You can't create invalid values — for example, BatterySoc is constrained to the 0–100% range
- You get consistent string representation across the application
- You can add domain-specific methods to value classes, e.g., rounding temperature values

### Protocols

AppDeamon support for testing is limited, so I had to extract logic from the framework-specific code and make it testable in isolation. I decided to use duck typing with `Protocol` classes to define AppDaemon interfaces, for example:

```python
class AppdaemonService(Protocol):
    def call_service(self, service: str, **data) -> object: ...
    
class AppdaemonState(Protocol):
    def get_state(self, entity_id: str, attribute: str | None = None) -> object: ...

class AppdaemonLogger(Protocol):
    def log(self, msg: str, *args, level: str | int = logging.INFO) -> None: ...
```

Those three protocols are all I needed to interact with AppDaemon framework in my applications!
The protocols initialization is straightforward because the protocols match the AppDaemon API:

```python
import appdaemon.plugins.hass.hassapi as hass

class HvacApp(hass.Hass):
    def initialize(self) -> None:
        appdaemon_logger = self
        appdaemon_state = self
        appdaemon_service = self
        (...)
```

For testing I define mock implementations of those protocols in the top level `conftest.py` file.
The fixtures are automatically available in all test modules without the need to monkey-patch AppDaemon classes.

```python
@pytest.fixture
def mock_appdaemon_logger() -> Mock:
    return Mock()


@pytest.fixture
def mock_appdaemon_state() -> Mock:
    return Mock()


@pytest.fixture
def mock_appdaemon_service() -> Mock:
    return Mock()
```

### For comprehensions

I love Scala's for-comprehensions, and I also enjoy their Python equivalent. I implemented separate classes for different consumption-forecast strategies and combined them with a composite class. Using a nested for clause inside a single list comprehension is an elegant way to implement the composite design pattern.

```python
class ConsumptionForecast(Protocol):
    def hourly(self, period_start: datetime, period_hours: int) -> list[HourlyConsumptionEnergy]: ...

class ConsumptionForecastComposite:
    def __init__(self, *components: ConsumptionForecast) -> None:
        self.components = components

    def hourly(self, period_start: datetime, period_hours: int) -> list[HourlyConsumptionEnergy]:
        return [item for component in self.components for item in component.hourly(period_start, period_hours)]
```

## Interesting algorithms

Algorithms have never been my strength but with help from GitHub Copilot and a basic grasp of intuition, math and physics, I implemented a few useful ones.

### Find the continuous time window with maximum revenue

This algorithm evaluates all possible starting minutes to find the optimal battery discharge window with maximum revenue.
It uses a variable-length sliding window approach with time complexity `O(n * m)` where `n` is number of periods and `m` is minutes per period.

```python
if max_duration_minutes < 1:
        raise ValueError(f"max_duration_minutes must be at least 1, got {max_duration_minutes}")

    if not hourly_prices:
        return None

    if not any(hourly_price.price >= min_price_threshold for hourly_price in hourly_prices):
        return None

    period_duration_minutes = 60

    max_revenue = None
    best_start_time = None
    best_end_time = None

    # Use variable-length sliding window algorithm, optimized two-pointer sliding window approach is not applicable here
    for start_hour_idx, start_hour in enumerate(hourly_prices):
        # Skip if period doesn't meet price threshold
        if start_hour.price < min_price_threshold:
            continue

        # Try starting at each minute within this period
        for start_offset_minutes in range(period_duration_minutes):
            start_time = start_hour.period.start + timedelta(minutes=start_offset_minutes)

            # Calculate revenue for a window of max_duration_minutes from this start
            revenue = start_hour.price.zeroed()
            minutes_covered = 0

            # Iterate through periods that this window spans
            current_period_idx = start_hour_idx
            minutes_into_current_period = start_offset_minutes

            while minutes_covered < max_duration_minutes and current_period_idx < len(hourly_prices):
                current_hourly_price = hourly_prices[current_period_idx]

                # Check threshold - stop if this period doesn't meet it
                if current_hourly_price.price < min_price_threshold:
                    break

                # Calculate how many minutes to take from this period
                minutes_available_in_period = period_duration_minutes - minutes_into_current_period
                minutes_needed = max_duration_minutes - minutes_covered
                minutes_to_take = min(minutes_available_in_period, minutes_needed)

                # Add revenue
                price_per_minute = current_hourly_price.price / Decimal(period_duration_minutes)
                revenue += price_per_minute * Decimal(minutes_to_take)
                minutes_covered += minutes_to_take

                # Move to next period
                current_period_idx += 1
                minutes_into_current_period = 0

            # Check if this is a valid and better solution
            if minutes_covered >= 1:
                end_time = start_time + timedelta(minutes=minutes_covered)

                if max_revenue is None or revenue > max_revenue:
                    max_revenue = revenue
                    best_start_time = start_time
                    best_end_time = end_time

    if max_revenue is None or best_start_time is None or best_end_time is None:
        return None

    return (max_revenue, best_start_time, best_end_time)
```

For the following hourly prices:

```python
[
    ("00:00:00", 100),
    ("01:00:00", 150),
    ("02:00:00", 200),
    ("03:00:00", 120),
]
```

The maximum revenue for a 105 minutes window with minimum price threshold of 100 is `150 * 45 / 60 + 200 = 312.5`, starting at "01:15:00" and ending at "03:00:00".

### Estimate heating energy consumption

This function estimates the electrical energy consumption of a heat pump by accounting for:

- Heat loss proportional to indoor/outdoor temperature difference
- COP (Coefficient of Performance) variation with outdoor temperature
- COP degradation due to frosting cycles in specific temperature and humidity ranges

The model assumes linear heat loss and uses empirically-derived adjustments for
real-world heat pump performance characteristics.

For example, for the outdoor temperature of 3.5°C, indoor temperature of 20°C, humidity of 100% with frosting penalty, COP at 7°C of 4.0, and heat loss coefficient of 0.18 kW/°C, the estimated heating energy consumption is approximately 0.987 kWh.

```python
_REFERENCE_TEMPERATURE = Celsius(7.0)  # temperature at which COP is rated
_COP_TEMPERATURE_COEFFICIENT = 0.033  # COP change per degree Celsius
_COP_COEFFICIENT_MIN = 0.5  # minimum COP multiplier to prevent unrealistic values

# Frosting cycle parameters
_FROSTING_TEMP_MIN = Celsius(0.0)  # lower bound for frosting conditions
_FROSTING_TEMP_MAX = Celsius(7.0)  # upper bound for frosting conditions
_FROSTING_TEMP_PEAK = Celsius(3.5)  # temperature with maximum frosting risk
_FROSTING_HUMIDITY_THRESHOLD = 70.0  # % - minimum humidity for frosting
_FROSTING_PENALTY_MAX = 0.15  # maximum COP reduction due to frosting cycles (15%)


def estimate_heating_energy_consumption(
    t_out: Celsius,
    t_in: Celsius,
    humidity: float,
    cop_at_7c: float,
    h: float,
) -> EnergyKwh:
    t_diff = t_in - t_out

    if t_diff.value > 0:
        heat_loss = h * t_diff.value

        temperature_coefficient = _temperature_coefficient(t_out)
        frosting_penalty = _frosting_penalty(t_out, humidity)
        adjusted_cop = cop_at_7c * temperature_coefficient * frosting_penalty

        energy_consumption = EnergyKwh(heat_loss / adjusted_cop)
    else:
        energy_consumption = ENERGY_KWH_ZERO

    return energy_consumption


def _temperature_coefficient(
    t_out: Celsius,
) -> float:
    temp_delta = t_out - _REFERENCE_TEMPERATURE
    coefficient = 1.0 + (temp_delta.value * _COP_TEMPERATURE_COEFFICIENT)

    return max(_COP_COEFFICIENT_MIN, coefficient)


def _frosting_penalty(
    t_out: Celsius,
    humidity: float,
) -> float:
    if not (_FROSTING_TEMP_MIN <= t_out <= _FROSTING_TEMP_MAX and humidity > _FROSTING_HUMIDITY_THRESHOLD):
        return 1.0

    distance_from_peak = abs(t_out.value - _FROSTING_TEMP_PEAK.value)
    frosting_severity = (_FROSTING_TEMP_PEAK.value - distance_from_peak) / _FROSTING_TEMP_PEAK.value

    humidity_factor = (humidity - _FROSTING_HUMIDITY_THRESHOLD) / (100.0 - _FROSTING_HUMIDITY_THRESHOLD)

    penalty = _FROSTING_PENALTY_MAX * frosting_severity * humidity_factor

    return 1.0 - penalty
```

### Calculate maximum cumulative energy deficit

To calculate the battery reserve SOC in the morning, I need to find the maximum cumulative energy deficit before next low tariff period. For example, to survive this hypothetical morning the battery needs to cover 1.25 kWh deficit:

| hour | production (kWh) | consumption (kWh) | cumulative deficit (kWh) |
|------|------------------|-------------------|--------------------------|
| 7    | 0.25             | 0.5               | 0.25                     |
| 8    | 1.0              | 2.0               | **1.25**                 |
| 9    | 2.0              | 1.0               | 0.25                     |
| 10   | 0.25             | 0.5               | 0.5                      |

```python
def maximum_cumulative_deficit(
    consumptions: list[HourlyConsumptionEnergy], productions: list[HourlyProductionEnergy]
) -> EnergyKwh:
    net_energy_dict = {}

    for consumption in consumptions:
        if consumption.period in net_energy_dict:
            net_energy_dict[consumption.period] -= consumption.energy
        else:
            net_energy_dict[consumption.period] = -consumption.energy

    for production in productions:
        if production.period in net_energy_dict:
            net_energy_dict[production.period] += production.energy
        else:
            net_energy_dict[production.period] = production.energy

    net_energy_list = list(net_energy_dict.items())
    net_energy_list_sorted = sorted(net_energy_list, key=lambda n: n[0].start)

    cumulative_balance = ENERGY_KWH_ZERO
    min_cumulative_balance = ENERGY_KWH_ZERO

    for net_energy in net_energy_list_sorted:
        cumulative_balance = cumulative_balance + net_energy[1]

        if cumulative_balance < min_cumulative_balance:
            min_cumulative_balance = cumulative_balance

    return max(-min_cumulative_balance, ENERGY_KWH_ZERO)
```

### Estimate time for indoor temperature to decay from start to end temperature

This function estimates the time required for indoor temperature to decay from a starting temperature to an ending temperature, considering varying outdoor temperatures over time. It uses Newton's Law of Cooling with a decay rate constant derived from the building's thermal properties.

```python
def estimate_temperature_decay_time(
    temp_start: Celsius,
    temp_end: Celsius,
    hourly_weather: list[HourlyWeather],
    decay_rate: float,
) -> timedelta:
    if decay_rate <= 0:
        raise ValueError("Decay rate must be positive")

    temp_current = temp_start
    temp_target = temp_end

    total_hours = 0.0

    for weather in hourly_weather:
        temp_outdoor = weather.temperature

        if temp_current <= temp_target:
            break

        if temp_current <= temp_outdoor:
            break

        temp_diff_start = temp_current - temp_outdoor
        temp_after_hour = temp_outdoor + Celsius(temp_diff_start.value * exp(-decay_rate * 1.0))

        if temp_after_hour <= temp_target:
            fraction = log(temp_diff_start / (temp_target - temp_outdoor)) / decay_rate
            total_hours += fraction
            break

        temp_current = temp_after_hour
        total_hours += 1.0

    return timedelta(hours=total_hours)
```

For example, starting from 22.9°C to 21.3°C with constant outdoor temperature of 8.0°C and decay rate of 0.0226, the estimated time is approximately 5.03 hours.

## Summary

I had a lot of fun building this project and learning Python along the way.
During implementation I deliberately avoided some language features from the book like: multiple inheritance and generics, because I'm not a big fan.
I still plan to explore coroutines, async/await, and metaclasses in more depth.

Getting back to the point, since the initial deployment in early autumn:

- The house stays warm and cozy.
- My family hasn't complained about hot water.
- Most grid consumption now happens in low‑tariff periods: 90% of grid energy is used then (total cost 0.58 PLN/kWh), while only 10% is used during high‑tariff periods (total cost 1.06 PLN/kWh).
- Self‑consumption of solar energy was 59% in October and 70% in November.

Don't forget to add a ⭐️ to my project on GitHub if you find it useful!
<https://github.com/mkuthan/home-assistant-appdaemon>
