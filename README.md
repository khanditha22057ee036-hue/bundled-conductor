# bundled-conductor
"""
Bundled Conductor Calculator
----------------------------
A menu-driven program for engineering students to calculate the GMR,
inductance, capacitance and surface voltage gradient of a three-phase
transmission line that uses bundled conductors (2 to 6 sub-conductors per phase).

Why bundle conductors?
    Bundling increases the effective radius of the phase conductor, which
        - reduces inductance and reactance (better power transfer)
        - increases capacitance (higher charging current)
        - reduces the surface voltage gradient (less corona loss and radio noise)

Key relations (sub-conductors on a circle, bundle spacing d):
    Bundle circle radius       R = d / (2 * sin(pi / n))
    Self GMD of one conductor  Ds = 0.7788 * r        (solid round conductor)
    Bundle GMR (inductance)    Ds_b = (n * Ds * R^(n-1))^(1/n)
    Bundle radius (capacitance) r_b = (n * r * R^(n-1))^(1/n)
    (n = 2: sqrt(Ds*d),   n = 3: (Ds*d^2)^(1/3),   n = 4: 1.09*(Ds*d^3)^(1/4))

Three-phase line (transposed), spacings D12, D23, D31:
    GMD = (D12 * D23 * D31)^(1/3)
    Inductance per phase      L = 2e-7 * ln(GMD / Ds_b)           (H/m)
    Capacitance per phase     C = 2*pi*e0 / ln(GMD / r_b)         (F/m)
    Inductive reactance       XL = 2*pi*f*L
    Capacitive reactance      XC = 1 / (2*pi*f*C)
    Charging current          Ic = 2*pi*f*C*V_phase

Surface voltage gradient (rms, on a sub-conductor):
    Average gradient          g_avg = V_phase / (n * r * ln(GMD / r_b))
    Maximum gradient          g_max = g_avg * (1 + (n - 1) * r / R)
    Corona starts when g_max reaches about 21.1 kV/cm (rms, standard air)
"""

import math

E0 = 8.854e-12            # permittivity of free space (F/m)
CORONA_GRADIENT = 21.1    # disruptive gradient of air (kV/cm, rms)


# ---------------------------------------------------------------- input helpers
def get_positive_float(prompt):
    """Keep asking until the user enters a valid positive number."""
    while True:
        try:
            value = float(input(prompt))
            if value <= 0:
                print("  Please enter a value greater than zero.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_int_in_range(prompt, low, high):
    """Ask for a whole number between low and high (inclusive)."""
    while True:
        try:
            value = int(input(prompt))
            if low <= value <= high:
                return value
            print(f"  Please enter a whole number from {low} to {high}.")
        except ValueError:
            print("  Invalid input. Please enter a whole number.")


def get_bundle(allow_single=True):
    """Ask for the bundle details. Returns (n, r_m, d_m)."""
    low = 1 if allow_single else 2
    n = get_int_in_range(f"Sub-conductors per phase n ({low} to 6): ", low, 6)
    r_cm = get_positive_float("Sub-conductor radius r (cm): ")
    d_m = 0.0
    if n > 1:
        d_m = get_positive_float("Bundle spacing d (cm): ") / 100
    return n, r_cm / 100, d_m


def get_spacing():
    """Ask for the phase spacings and return GMD in metres."""
    print("  Phase spacings (m). For equilateral spacing enter the same value three times.")
    d12 = get_positive_float("  D12 (m): ")
    d23 = get_positive_float("  D23 (m): ")
    d31 = get_positive_float("  D31 (m): ")
    return (d12 * d23 * d31) ** (1 / 3)


# ------------------------------------------------------------------ core maths
def circle_radius(n, d):
    """Radius of the circle through the sub-conductor centres."""
    return 0.0 if n == 1 else d / (2 * math.sin(math.pi / n))


def self_gmd(r):
    """Ds = 0.7788 * r for a solid round conductor."""
    return 0.7788 * r


def bundle_equivalent(n, base, d):
    """(n * base * R^(n-1))^(1/n); use base = Ds for GMR, base = r for radius."""
    if n == 1:
        return base
    big_r = circle_radius(n, d)
    return (n * base * big_r ** (n - 1)) ** (1 / n)


def inductance_per_km(gmd, ds_b):
    """L in mH/km  =  0.2 * ln(GMD / Ds_b)."""
    return 2e-7 * math.log(gmd / ds_b) * 1e6


def capacitance_per_km(gmd, r_b):
    """C in microfarad/km."""
    return 2 * math.pi * E0 / math.log(gmd / r_b) * 1e3 * 1e6


def surface_gradients(v_ph_kv, n, r, d, gmd, r_b):
    """Return (average, maximum) surface gradient in kV/cm (rms)."""
    g_avg = v_ph_kv / (n * (r * 100) * math.log(gmd / r_b))
    big_r = circle_radius(n, d)
    g_max = g_avg * (1 + (n - 1) * r / big_r) if n > 1 else g_avg
    return g_avg, g_max


def line_constants(n, r, d, gmd):
    """Return GMR, bundle radius, L (mH/km), C (uF/km) for the given bundle."""
    ds_b = bundle_equivalent(n, self_gmd(r), d)
    r_b = bundle_equivalent(n, r, d)
    return ds_b, r_b, inductance_per_km(gmd, ds_b), capacitance_per_km(gmd, r_b)


# ------------------------------------------------------------------- display
def bundle_gmr_option():
    n, r, d = get_bundle(allow_single=False)
    ds = self_gmd(r)
    ds_b = bundle_equivalent(n, ds, d)
    r_b = bundle_equivalent(n, r, d)

    print("\n  ----- Bundle GMR and Equivalent Radius -----")
    print(f"  Sub-conductor self GMD Ds   = {ds * 100:.4f} cm")
    print(f"  Bundle circle radius R      = {circle_radius(n, d) * 100:.3f} cm")
    print(f"  Bundle GMR (inductance) Ds_b = {ds_b * 100:.4f} cm")
    print(f"  Bundle radius (capacitance) r_b = {r_b * 100:.4f} cm")
    print(f"  Ds_b is {ds_b / ds:.2f} times the single-conductor GMR")


def inductance_option():
    n, r, d = get_bundle()
    gmd = get_spacing()
    f = get_positive_float("Frequency f (Hz): ")
    length = get_positive_float("Line length (km): ")

    ds_b, _, l_km, _ = line_constants(n, r, d, gmd)
    xl_km = 2 * math.pi * f * l_km / 1000

    print("\n  ----- Inductance and Reactance -----")
    print(f"  GMD between phases      = {gmd:.3f} m")
    print(f"  Bundle GMR Ds_b         = {ds_b * 100:.4f} cm")
    print(f"  Inductance per phase    = {l_km:.4f} mH/km  ({l_km * length:.3f} mH total)")
    print(f"  Reactance per phase     = {xl_km:.4f} ohm/km  ({xl_km * length:.3f} ohm total)")


def capacitance_option():
    n, r, d = get_bundle()
    gmd = get_spacing()
    f = get_positive_float("Frequency f (Hz): ")
    length = get_positive_float("Line length (km): ")
    v_kv = get_positive_float("Line voltage (kV): ")

    _, r_b, _, c_km = line_constants(n, r, d, gmd)
    xc_km = 1 / (2 * math.pi * f * c_km * 1e-6) / 1000       # kilo-ohm x km -> ohm-km basis
    b_km = 2 * math.pi * f * c_km                            # microsiemens/km
    i_charge = 2 * math.pi * f * c_km * 1e-6 * length * (v_kv * 1000 / math.sqrt(3))

    print("\n  ----- Capacitance and Charging Current -----")
    print(f"  GMD between phases      = {gmd:.3f} m")
    print(f"  Bundle radius r_b       = {r_b * 100:.4f} cm")
    print(f"  Capacitance to neutral  = {c_km:.5f} microfarad/km  ({c_km * length:.3f} microfarad total)")
    print(f"  Shunt susceptance       = {b_km:.4f} microsiemens/km")
    print(f"  Capacitive reactance    = {xc_km:.2f} kilo-ohm x km  ({xc_km / length:.3f} kilo-ohm for the line)")
    print(f"  Charging current        = {i_charge:.2f} A per phase")
    print(f"  Charging reactive power = {math.sqrt(3) * v_kv * i_charge / 1000:.3f} MVAR (three-phase)")


def comparison_option():
    n, r, d = get_bundle(allow_single=False)
    gmd = get_spacing()
    f = get_positive_float("Frequency f (Hz): ")
    v_kv = get_positive_float("Line voltage (kV): ")

    # Single conductor with the same total conductor cross-section
    r_single = r * math.sqrt(n)

    rows = {
        f"Single conductor (r = {r_single * 100:.2f} cm)": line_constants(1, r_single, 0, gmd),
        f"Bundle of {n} (r = {r * 100:.2f} cm each)": line_constants(n, r, d, gmd),
    }
    v_ph = v_kv / math.sqrt(3)

    print("\n  ----- Bundled vs Single Conductor (same copper/aluminium area) -----")
    print(f"  {'':38}{'L (mH/km)':>11}{'XL (ohm/km)':>13}{'C (uF/km)':>11}{'g_max (kV/cm)':>15}")
    values = []
    for label, (ds_b, r_b, l_km, c_km) in rows.items():
        n_eff = 1 if label.startswith("Single") else n
        r_eff = r_single if n_eff == 1 else r
        _, g_max = surface_gradients(v_ph, n_eff, r_eff, d, gmd, r_b)
        xl = 2 * math.pi * f * l_km / 1000
        values.append((l_km, c_km, g_max))
        print(f"  {label:38}{l_km:>11.4f}{xl:>13.4f}{c_km:>11.5f}{g_max:>15.2f}")

    (l1, c1, g1), (l2, c2, g2) = values
    print(f"\n  Bundling changes: inductance {(l2 / l1 - 1) * 100:+.1f} %, capacitance {(c2 / c1 - 1) * 100:+.1f} %,")
    print(f"  maximum surface gradient {(g2 / g1 - 1) * 100:+.1f} %.")


def corona_option():
    n, r, d = get_bundle()
    gmd = get_spacing()
    v_kv = get_positive_float("Line voltage (kV): ")

    _, r_b, _, _ = line_constants(n, r, d, gmd)
    v_ph = v_kv / math.sqrt(3)
    g_avg, g_max = surface_gradients(v_ph, n, r, d, gmd, r_b)
    margin = CORONA_GRADIENT / g_max

    print("\n  ----- Surface Gradient and Corona Check -----")
    print(f"  Phase voltage (rms)     = {v_ph:.2f} kV")
    print(f"  Average surface gradient = {g_avg:.2f} kV/cm (rms)")
    print(f"  Maximum surface gradient = {g_max:.2f} kV/cm (rms)")
    print(f"  Disruptive gradient of air = {CORONA_GRADIENT} kV/cm (rms)")
    if g_max >= CORONA_GRADIENT:
        print("  Result: Corona is LIKELY. Increase the conductor size, bundle or spacing.")
    else:
        print(f"  Result: Below the corona limit (safety factor {margin:.2f}).")
    print("  (Rain, dirt and rough surfaces lower the real limit, so a margin is needed.)")


def menu():
    print("\n" + "=" * 58)
    print("          BUNDLED CONDUCTOR CALCULATOR")
    print("=" * 58)
    print(" 1. Bundle GMR and equivalent radius")
    print(" 2. Inductance and inductive reactance")
    print(" 3. Capacitance, susceptance and charging current")
    print(" 4. Compare bundled vs single conductor")
    print(" 5. Surface voltage gradient and corona check")
    print(" 0. Exit")
    print("-" * 58)


def main():
    while True:
        menu()
        choice = input("Enter your choice: ").strip()

        if choice == "1":
            bundle_gmr_option()
        elif choice == "2":
            inductance_option()
        elif choice == "3":
            capacitance_option()
        elif choice == "4":
            comparison_option()
        elif choice == "5":
            corona_option()
        elif choice == "0":
            print("\nThank you for using the calculator. Goodbye!")
            break
        else:
            print("  Invalid choice. Please select from the menu.")


if __name__ == "__main__":
    main()
