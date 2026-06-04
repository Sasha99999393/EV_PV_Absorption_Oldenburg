# EV_PV_Absorption_Oldenburg
How many EVs are needed to absorb local PV generation at summer midday? This notebook calculates that for Oldenburg, Germany (1 May 2019, 10:00–16:00) using live Renewables.ninja solar data and a six-model EV fleet average. Run after adding your API token and local installed PV capacity.
import json
import requests
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings("ignore")

plt.rcParams.update({"figure.dpi": 120, "axes.spines.top": False,
                     "axes.spines.right": False, "font.size": 11})

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 1 — PV GENERATION DATA (Renewables.ninja API)
# Reference: Pfenninger & Staffell 2016 (R1); API docs (R2)
# ══════════════════════════════════════════════════════════════════════════════

# ► DATA INPUT 1: paste your token from https://www.renewables.ninja → Profile
API_TOKEN = "YOUR_RENEWABLES_NINJA_TOKEN_HERE"

LAT, LON    = 52.9302, 8.0522   # Oldenburg city centre
DATE        = "2019-05-01"      # 1 May 2019
TILT        = 35                # degrees — typical German rooftop
AZIM        = 180               # south-facing
SYSTEM_LOSS = 0.10              # 10 % system losses (Renewables.ninja default)

s = requests.Session()
s.headers.update({"Authorization": f"Token {API_TOKEN}"})

args = {
    "lat": LAT, "lon": LON,
    "date_from": DATE, "date_to": DATE,
    "dataset": "merra2", "capacity": 1.0,
    "system_loss": SYSTEM_LOSS, "tracking": 0,
    "tilt": TILT, "azim": AZIM,
    "local_time": "true", "format": "json",
}

print(f"Querying Renewables.ninja for Oldenburg on {DATE} ...")
response = s.get("https://www.renewables.ninja/api/data/pv", params=args, timeout=30)

parsed = json.loads(response.text)
df_pv = pd.read_json(json.dumps(parsed["data"]), orient="index")
df_pv.index = pd.to_datetime(df_pv.index)
df_pv = df_pv.rename(columns={"electricity": "cf"})   # cf = kW per kWp installed
print(f"✅ {len(df_pv)} hourly records | peak CF: {df_pv['cf'].max():.3f} kW/kWp")

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 2 — FILTER TO 10:00–16:00 WINDOW & PLOT
# Germany is in CEST (UTC+2) on 1 May; API returns local time
# ══════════════════════════════════════════════════════════════════════════════

mask      = (df_pv.index.hour >= 10) & (df_pv.index.hour <= 15)
df_window = df_pv[mask].copy()

energy_per_kwp_window = df_window["cf"].sum()   # kWh per kWp (1 h per row)
print(f"\nEnergy per kWp in 10–16h window : {energy_per_kwp_window:.4f} kWh/kWp")
print(df_window.to_string())

fig, ax = plt.subplots(figsize=(9, 4))
ax.bar(df_pv.index.hour, df_pv["cf"],
       color="gold", edgecolor="orange", alpha=0.7, label="Full day")
ax.bar(df_window.index.hour, df_window["cf"],
       color="darkorange", edgecolor="saddlebrown", label="10:00–16:00 window")
ax.axvspan(9.5, 15.5, alpha=0.07, color="green")
ax.set_xlabel("Hour (local CEST)")
ax.set_ylabel("Capacity Factor (kW/kWp)")
ax.set_title(f"PV Output — Oldenburg, 1 May 2019\nRenewables.ninja | MERRA-2 | 35° tilt, S-facing, 10% losses")
ax.set_xticks(range(24))
ax.legend()
plt.tight_layout()
plt.show()

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 3 — OLDENBURG INSTALLED PV CAPACITY
# Source: Bundesnetzagentur Marktstammdatenregister (R8)
# HOW TO GET: https://www.marktstammdatenregister.de
#   Filter: Energieträger → Solare Strahlungsenergie, Gemeinde → Oldenburg (Oldb)
#   Sum the Bruttoleistung column [kWp] and convert to MWp
# ► DATA INPUT 2: replace with value from MaStR
# ══════════════════════════════════════════════════════════════════════════════

INSTALLED_PV_MWP = 150.0   # ← MWp — replace with actual MaStR value
print(f"\nInstalled PV capacity (Oldenburg) : {INSTALLED_PV_MWP:.1f} MWp  [source: MaStR — R8]")

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 4 — TOTAL PV ENERGY IN WINDOW
# E_PV = CF_window [kWh/kWp] × installed capacity [kWp]
# ══════════════════════════════════════════════════════════════════════════════

installed_kwp = INSTALLED_PV_MWP * 1000
E_pv_kwh      = energy_per_kwp_window * installed_kwp
print(f"Total PV energy (10–16h)          : {E_pv_kwh:,.0f} kWh  ({E_pv_kwh/1000:.1f} MWh)")

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 5 — EV FLEET CHARACTERISATION
# Sources: ev-database.org (R3), EVspecs.org (R4), VW Newsroom (R5),
#          Motor1/InsideEVs (R6)
# ══════════════════════════════════════════════════════════════════════════════

ev_fleet = pd.DataFrame({
    "model": [
        "Tesla Model 3 RWD (Highland 2024)",
        "Tesla Model Y RWD (2024)",
        "VW ID.3 Pro S 77 kWh (MY24)",
        "VW ID.4 Pro S 77 kWh (MY24)",
        "BMW i4 eDrive40 (2024)",
        "BMW iX1 eDrive20 (2024)",
    ],
    "battery_usable_kwh": [60.5, 62.0, 77.0, 77.0, 81.3, 64.7],
    "ac_obc_kw":          [11,   11,   11,   11,   11,   11  ],
})

print("\nEV fleet specs:")
print(ev_fleet.to_string(index=False))

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 6 — AVERAGE EV & ENERGY ABSORBABLE PER VEHICLE
# SOC window 20→80%; AC efficiency 90% (Lecture 7 slide 30 — R7)
# Binding constraint = min(SOC headroom, charger limit over 6 h)
# ══════════════════════════════════════════════════════════════════════════════

SOC_DELTA    = 0.60   # 80% − 20%
WINDOW_HOURS = 6
ETA_CHARGE   = 0.90   # AC wallbox-to-battery efficiency (R7)

ev_fleet["energy_soc_kwh"]     = ev_fleet["battery_usable_kwh"] * SOC_DELTA
ev_fleet["energy_charger_kwh"] = ev_fleet["ac_obc_kw"] * WINDOW_HOURS * ETA_CHARGE
ev_fleet["energy_absorb_kwh"]  = ev_fleet[["energy_soc_kwh",
                                            "energy_charger_kwh"]].min(axis=1)

avg_usable = ev_fleet["battery_usable_kwh"].mean()
avg_obc    = ev_fleet["ac_obc_kw"].mean()
avg_absorb = ev_fleet["energy_absorb_kwh"].mean()

print(f"\nAverage EV — usable battery : {avg_usable:.1f} kWh")
print(f"Average EV — AC OBC power   : {avg_obc:.0f} kW")
print(f"Average EV — absorb in 6 h  : {avg_absorb:.2f} kWh")

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 7 — MAIN RESULT
# N = E_PV [kWh] / E_per_vehicle [kWh]
# ══════════════════════════════════════════════════════════════════════════════

N_ev = E_pv_kwh / avg_absorb

print("\n" + "═" * 55)
print("  RESULT")
print("═" * 55)
print(f"  Total PV energy in window  : {E_pv_kwh:>12,.1f} kWh")
print(f"  Energy per average EV      : {avg_absorb:>12.2f} kWh")
print(f"  EVs required (theoretical) : {N_ev:>12,.0f}")
print(f"  EVs required (rounded up)  : {int(np.ceil(N_ev)):>12,}")
print("═" * 55)

bev_fleet = int(170_000 * 0.55 * 0.0335)   # ~3,100; KBA Jan 2025 rate (R9, R7)
print(f"\nOldenburg current BEV stock (est.) : {bev_fleet:,}")
print(f"Required / current BEV stock       : {N_ev/bev_fleet:.1f}×")

# ══════════════════════════════════════════════════════════════════════════════
# SECTION 8 — SENSITIVITY ANALYSIS
# ══════════════════════════════════════════════════════════════════════════════

fig, axes = plt.subplots(1, 3, figsize=(14, 4))

pv_range  = np.linspace(50, 400, 200)
n_ev_pv   = (energy_per_kwp_window * pv_range * 1000) / avg_absorb
axes[0].plot(pv_range, n_ev_pv, color="orange", lw=2)
axes[0].axvline(INSTALLED_PV_MWP, ls="--", color="gray",
                label=f"Base: {INSTALLED_PV_MWP} MWp")
axes[0].set_xlabel("Installed PV capacity (MWp)")
axes[0].set_ylabel("EVs needed")
axes[0].set_title("Sensitivity: PV capacity")
axes[0].legend()

obc_range   = np.linspace(3.7, 22, 200)
absorb_obc  = np.minimum(avg_usable * SOC_DELTA, obc_range * WINDOW_HOURS * ETA_CHARGE)
n_ev_obc    = E_pv_kwh / absorb_obc
axes[1].plot(obc_range, n_ev_obc, color="steelblue", lw=2)
axes[1].axvline(avg_obc, ls="--", color="gray", label=f"Base: {avg_obc:.0f} kW")
axes[1].set_xlabel("AC OBC power (kW)")
axes[1].set_ylabel("EVs needed")
axes[1].set_title("Sensitivity: charger power")
axes[1].legend()

soc_range   = np.linspace(0.1, 0.9, 200)
absorb_soc  = np.minimum(avg_usable * soc_range,
                         avg_obc * WINDOW_HOURS * ETA_CHARGE)
n_ev_soc    = E_pv_kwh / absorb_soc
axes[2].plot(soc_range * 100, n_ev_soc, color="green", lw=2)
axes[2].axvline(SOC_DELTA * 100, ls="--", color="gray",
                label=f"Base: {SOC_DELTA*100:.0f}%")
axes[2].set_xlabel("Available SOC window (%)")
axes[2].set_ylabel("EVs needed")
axes[2].set_title("Sensitivity: SOC window")
axes[2].legend()

for ax in axes:
    ax.ticklabel_format(style="sci", axis="y", scilimits=(3, 3))

plt.suptitle("Sensitivity Analysis — EVs to absorb PV generation (Oldenburg, 1 May)",
             y=1.02)
plt.tight_layout()
plt.show()
