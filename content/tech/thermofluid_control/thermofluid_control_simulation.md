---
title: "Thermofluid Control Simulation"
date: 2025-02-02T20:10:14+09:00
cover:
    image: images/20250202_ns_equation.png # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
draft: false
---

# Here is my question to ChatGPT o3-mini high

Develop a program that simulates a thermal fluid system within a two-dimensional square region. Specify the boundary conditions for the four sides of the square area. The square region will calculate the thermo-fluid equation using the **Boussinesq approximation** to account for buoyancy. The program should compute the **flow rate, pressure, and temperature** within the square region.

Additionally, create a **control simulation** program that regulates the temperature of the square area. The program should accept a target temperature for the square area and set the temperature, flow rate, and pressure in the boundary conditions to **Neumann and Dirichlet conditions**. The program should also allow for the input of boundary conditions.

---

# Answer

Below is an example **Python program** that:

- Sets up a **two‐dimensional square domain**
- Solves the **thermal–fluid equations** (with buoyancy treated via the **Boussinesq approximation**)
- Provides a **control simulation** that accepts a **target temperature** and adjusts the boundary conditions accordingly.

> **Note:** This example uses a **finite–difference (FD) approach** on a uniform grid with an explicit time‐stepping method. For robust real-world simulations, you may use a more advanced method or a **CFD package**.



## Code Implementation

### Import Necessary Libraries
```python
#!/usr/bin/env python3
"""
Thermal–Fluid Simulation in a 2D Square Domain with Buoyancy (Boussinesq Approximation)
and a Control Simulation to regulate the temperature.

The simulation solves:
   • Continuity: ∇·u = 0
   • Momentum:
         du/dt + u·∇u = -1/ρ ∇p + ν ∇²u
         dv/dt + u·∇v = -1/ρ ∂p/∂y + ν ∇²v + g·β (T - T₀)
   • Energy (Temperature):
         dT/dt + u·∇T = α ∇²T

Boundary conditions on the four sides of the square can be either Dirichlet (fixed value)
or Neumann (fixed flux, approximated via a one–sided difference).

The “control_simulation” routine accepts a target temperature and resets the boundary
conditions so that, for example, the entire boundary is maintained at the target temperature.
"""

import numpy as np
import matplotlib.pyplot as plt

# ----------------------------
# Helper: Pressure Poisson Solver
# ----------------------------
def pressure_poisson(p, dx, dy, b, nit):
    """
    Solves the pressure Poisson equation using an iterative (Jacobi) method.
    
    Parameters:
        p   : pressure array
        dx  : grid spacing in x
        dy  : grid spacing in y
        b   : source term array (related to the velocity divergence)
        nit : number of iterations
    Returns:
        p : updated pressure field
    """
    for it in range(nit):
        pn = p.copy()
        p[1:-1, 1:-1] = (((pn[1:-1, 2:] + pn[1:-1, 0:-2]) * dy**2 +
                          (pn[2:, 1:-1] + pn[0:-2, 1:-1]) * dx**2 -
                          b[1:-1, 1:-1] * dx**2 * dy**2) /
                         (2 * (dx**2 + dy**2)))
        # For simplicity, here we apply homogeneous Neumann BCs on all sides:
        p[:, 0] = p[:, 1]       # left: dp/dx = 0
        p[:, -1] = p[:, -2]     # right: dp/dx = 0
        p[0, :] = p[1, :]       # top: dp/dy = 0
        p[-1, :] = p[-2, :]     # bottom: dp/dy = 0
    return p

# ----------------------------
# Helper: Apply Boundary Conditions
# ----------------------------
def apply_boundary_conditions(u, v, p, T, bc, dx, dy):
    """
    Applies the boundary conditions (BCs) to the fields.
    The bc dictionary should have the keys 'left', 'right', 'top', and 'bottom'.
    For each side, a sub–dictionary should be provided with entries for each variable,
    e.g., bc['left'] = { 'u': ('dirichlet', value), 'v': ('dirichlet', value),
                         'T': ('dirichlet', value), 'p': ('dirichlet', value) }.
                         
    For a Neumann condition we approximate:
         value at boundary = value at adjacent point + (flux * grid spacing)
    """
    # Left boundary (x = 0)
    if 'left' in bc:
        for var, (cond, val) in bc['left'].items():
            if var == 'u':
                if cond == 'dirichlet':
                    u[:, 0] = val
                elif cond == 'neumann':
                    u[:, 0] = u[:, 1] + val * dx
            elif var == 'v':
                if cond == 'dirichlet':
                    v[:, 0] = val
                elif cond == 'neumann':
                    v[:, 0] = v[:, 1] + val * dx
            elif var == 'T':
                if cond == 'dirichlet':
                    T[:, 0] = val
                elif cond == 'neumann':
                    T[:, 0] = T[:, 1] + val * dx
            elif var == 'p':
                if cond == 'dirichlet':
                    p[:, 0] = val
                elif cond == 'neumann':
                    p[:, 0] = p[:, 1] + val * dx

    # Right boundary (x = L)
    if 'right' in bc:
        for var, (cond, val) in bc['right'].items():
            if var == 'u':
                if cond == 'dirichlet':
                    u[:, -1] = val
                elif cond == 'neumann':
                    u[:, -1] = u[:, -2] + val * dx
            elif var == 'v':
                if cond == 'dirichlet':
                    v[:, -1] = val
                elif cond == 'neumann':
                    v[:, -1] = v[:, -2] + val * dx
            elif var == 'T':
                if cond == 'dirichlet':
                    T[:, -1] = val
                elif cond == 'neumann':
                    T[:, -1] = T[:, -2] + val * dx
            elif var == 'p':
                if cond == 'dirichlet':
                    p[:, -1] = val
                elif cond == 'neumann':
                    p[:, -1] = p[:, -2] + val * dx

    # Top boundary (y = 0)
    if 'top' in bc:
        for var, (cond, val) in bc['top'].items():
            if var == 'u':
                if cond == 'dirichlet':
                    u[0, :] = val
                elif cond == 'neumann':
                    u[0, :] = u[1, :] + val * dy
            elif var == 'v':
                if cond == 'dirichlet':
                    v[0, :] = val
                elif cond == 'neumann':
                    v[0, :] = v[1, :] + val * dy
            elif var == 'T':
                if cond == 'dirichlet':
                    T[0, :] = val
                elif cond == 'neumann':
                    T[0, :] = T[1, :] + val * dy
            elif var == 'p':
                if cond == 'dirichlet':
                    p[0, :] = val
                elif cond == 'neumann':
                    p[0, :] = p[1, :] + val * dy

    # Bottom boundary (y = L)
    if 'bottom' in bc:
        for var, (cond, val) in bc['bottom'].items():
            if var == 'u':
                if cond == 'dirichlet':
                    u[-1, :] = val
                elif cond == 'neumann':
                    u[-1, :] = u[-2, :] + val * dy
            elif var == 'v':
                if cond == 'dirichlet':
                    v[-1, :] = val
                elif cond == 'neumann':
                    v[-1, :] = v[-2, :] + val * dy
            elif var == 'T':
                if cond == 'dirichlet':
                    T[-1, :] = val
                elif cond == 'neumann':
                    T[-1, :] = T[-2, :] + val * dy
            elif var == 'p':
                if cond == 'dirichlet':
                    p[-1, :] = val
                elif cond == 'neumann':
                    p[-1, :] = p[-2, :] + val * dy

    return u, v, p, T

# ----------------------------
# Main Simulation Function
# ----------------------------
def simulate_thermal_fluid(nx=50, ny=50, nt=500, nit=50, dt=0.001, L=1.0,
                           nu=0.1, alpha=0.01, rho=1.0, g=9.81, beta=0.1, T0=300.0,
                           bc=None):
    """
    Simulates the thermal–fluid system in a square domain.
    
    Parameters:
        nx, ny : Number of grid points in x and y
        nt     : Number of time steps
        nit    : Number of iterations for the pressure Poisson solver
        dt     : Time step
        L      : Length of the domain (assumed square)
        nu     : Kinematic viscosity
        alpha  : Thermal diffusivity
        rho    : Fluid density
        g      : Acceleration due to gravity
        beta   : Thermal expansion coefficient (buoyancy factor)
        T0     : Reference temperature (background temperature)
        bc     : Dictionary containing boundary conditions for each side.
                 (If None, a default set of BCs is used.)
                 
    Returns:
        x, y       : 1D arrays for the grid in x and y directions
        u, v, p, T : 2D arrays for the velocity components, pressure, and temperature
        flow_rate  : An estimate of the net flow rate (integral of u)
    """
    # Compute grid spacing and define grid arrays
    dx = L / (nx - 1)
    dy = L / (ny - 1)
    x = np.linspace(0, L, nx)
    y = np.linspace(0, L, ny)

    # Initialize the fields
    u = np.zeros((ny, nx))         # x–velocity
    v = np.zeros((ny, nx))         # y–velocity
    p = np.zeros((ny, nx))         # Pressure
    T = np.ones((ny, nx)) * T0     # Temperature field (initialized to reference temperature)

    # If no boundary condition dictionary is provided, use default BCs.
    if bc is None:
        # Example default BCs:
        #   • Left boundary (x = 0): Inflow with u=1, v=0 and an elevated temperature (T0+50)
        #   • Right boundary (x = L): Outflow with zero–gradient for velocity and temperature
        #   • Top and bottom (y = 0 and y = L): No–slip walls with fixed temperature T0
        bc = {
            'left':   {'u': ('dirichlet', 1.0),
                       'v': ('dirichlet', 0.0),
                       'T': ('dirichlet', T0 + 50),
                       'p': ('dirichlet', 0.0)},
            'right':  {'u': ('neumann', 0.0),
                       'v': ('neumann', 0.0),
                       'T': ('neumann', 0.0),
                       'p': ('dirichlet', 0.0)},
            'top':    {'u': ('dirichlet', 0.0),
                       'v': ('dirichlet', 0.0),
                       'T': ('dirichlet', T0),
                       'p': ('neumann', 0.0)},
            'bottom': {'u': ('dirichlet', 0.0),
                       'v': ('dirichlet', 0.0),
                       'T': ('dirichlet', T0),
                       'p': ('neumann', 0.0)}
        }

    # Time-stepping loop
    for n in range(nt):
        un = u.copy()
        vn = v.copy()
        Tn = T.copy()

        # Compute the source term for the pressure Poisson equation (related to continuity)
        b = np.zeros((ny, nx))
        b[1:-1, 1:-1] = (rho * (1/dt) *
                         ((un[1:-1, 2:] - un[1:-1, 0:-2]) / (2 * dx) +
                          (vn[2:, 1:-1] - vn[0:-2, 1:-1]) / (2 * dy)))

        # --- Update the temperature field (Energy Equation) ---
        T[1:-1, 1:-1] = (Tn[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (Tn[1:-1, 1:-1] - Tn[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (Tn[1:-1, 1:-1] - Tn[0:-2, 1:-1]) / dy) +
                         dt * alpha * ((Tn[1:-1, 2:] - 2 * Tn[1:-1, 1:-1] + Tn[1:-1, 0:-2]) / dx**2 +
                                       (Tn[2:, 1:-1] - 2 * Tn[1:-1, 1:-1] + Tn[0:-2, 1:-1]) / dy**2))

        # --- Update the u–momentum (x–direction) ---
        u[1:-1, 1:-1] = (un[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (un[1:-1, 1:-1] - un[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (un[1:-1, 1:-1] - un[0:-2, 1:-1]) / dy) -
                         dt * (1 / rho) * ((p[1:-1, 2:] - p[1:-1, 0:-2]) / (2 * dx)) +
                         dt * nu * ((un[1:-1, 2:] - 2 * un[1:-1, 1:-1] + un[1:-1, 0:-2]) / dx**2 +
                                    (un[2:, 1:-1] - 2 * un[1:-1, 1:-1] + un[0:-2, 1:-1]) / dy**2))

        # --- Update the v–momentum (y–direction) with buoyancy ---
        # The buoyancy term g*β*(T - T₀) is added here (Boussinesq approximation)
        v[1:-1, 1:-1] = (vn[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (vn[1:-1, 1:-1] - vn[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (vn[1:-1, 1:-1] - vn[0:-2, 1:-1]) / dy) -
                         dt * (1 / rho) * ((p[2:, 1:-1] - p[0:-2, 1:-1]) / (2 * dy)) +
                         dt * nu * ((vn[1:-1, 2:] - 2 * vn[1:-1, 1:-1] + vn[1:-1, 0:-2]) / dx**2 +
                                    (vn[2:, 1:-1] - 2 * vn[1:-1, 1:-1] + vn[0:-2, 1:-1]) / dy**2) +
                         dt * g * beta * (Tn[1:-1, 1:-1] - T0))

        # --- Solve for pressure via the Poisson equation ---
        p = pressure_poisson(p, dx, dy, b, nit)

        # --- Enforce the boundary conditions on all fields ---
        u, v, p, T = apply_boundary_conditions(u, v, p, T, bc, dx, dy)

    # --- Compute an estimate of the flow rate ---
    # (For example, integrate u over the left boundary)
    flow_rate = np.sum(u[:, 0]) * dy

    return x, y, u, v, p, T, flow_rate

# ----------------------------
# Control Simulation Function
# ----------------------------
def control_simulation(target_temperature, custom_bc=None, **kwargs):
    """
    Runs a simulation in which the boundary conditions are set to regulate the
    temperature of the square region. In this control simulation the temperature
    on every boundary is fixed (Dirichlet) to the target_temperature. Optionally,
    the user may supply a custom boundary condition dictionary (custom_bc) to override
    some values.
    
    Parameters:
        target_temperature : Desired temperature (e.g., in Kelvin) to maintain on boundaries.
        custom_bc          : (Optional) Dictionary with boundary conditions to override defaults.
        **kwargs           : Additional parameters passed to simulate_thermal_fluid.
    
    Returns:
        x, y, u, v, p, T, flow_rate : Simulation results.
    """
    # Define a default control BC dictionary.
    # In this example we fix the temperature on every side to the target_temperature.
    bc_control = {
        'left':   {'u': ('dirichlet', 1.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', target_temperature),
                   'p': ('dirichlet', 0.0)},
        'right':  {'u': ('neumann', 0.0),
                   'v': ('neumann', 0.0),
                   'T': ('dirichlet', target_temperature),
                   'p': ('dirichlet', 0.0)},
        'top':    {'u': ('dirichlet', 0.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', target_temperature),
                   'p': ('neumann', 0.0)},
        'bottom': {'u': ('dirichlet', 0.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', target_temperature),
                   'p': ('neumann', 0.0)}
    }
    # If the user supplies any custom boundary conditions, update the default.
    if custom_bc is not None:
        for side in custom_bc:
            bc_control[side].update(custom_bc[side])
            
    # Run the simulation using the control BCs.
    return simulate_thermal_fluid(bc=bc_control, **kwargs)

# ----------------------------
# Main: Run Simulations and Plot Results
# ----------------------------
if __name__ == "__main__":
    # --- Run a Standard Simulation ---
    x, y, u, v, p, T, flow_rate = simulate_thermal_fluid()
    print("Standard Simulation:")
    print("  Flow rate estimate =", flow_rate)
    
    # Plot the temperature field from the standard simulation
    plt.figure(figsize=(6, 5))
    cp = plt.contourf(x, y, T, 20, cmap='hot')
    plt.colorbar(cp)
    plt.title("Temperature Field (Standard Simulation)")
    plt.xlabel("x")
    plt.ylabel("y")
    plt.tight_layout()
    plt.show()
    
    # --- Run a Control Simulation ---
    target_temp = 320.0  # For example, regulate the boundary to 320 K
    x, y, u, v, p, T, flow_rate = control_simulation(target_temperature=target_temp)
    print("\nControl Simulation (Target Temperature = {} K):".format(target_temp))
    print("  Flow rate estimate =", flow_rate)
    
    # Plot the temperature field from the control simulation
    plt.figure(figsize=(6, 5))
    cp = plt.contourf(x, y, T, 20, cmap='hot')
    plt.colorbar(cp)
    plt.title("Temperature Field (Control Simulation)")
    plt.xlabel("x")
    plt.ylabel("y")
    plt.tight_layout()
    plt.show()
```

Explanation

1. Governing Equations & Boussinesq Term
	•	The momentum equations include buoyancy via:

```python
dt * g * beta * (Tn[1:-1, 1:-1] - T0)
```


2. Boundary Conditions
	•	A dictionary bc defines conditions for temperature, velocity, and pressure.
	•	The function apply_boundary_conditions() enforces these after each time step.

3. Control Simulation
	•	The function control_simulation() modifies boundary conditions so that temperature remains constant at a user-defined value.

4. Running and Plotting
	•	The script runs a standard and controlled simulation, then plots the temperature field.

Summary
	•	This Python simulation serves as a template for thermo-fluid control simulations.
	•	The program uses finite-difference methods with Boussinesq approximation.
	•	Users can modify boundary conditions, grid resolution, and time steps.

Happy simulating! 🚀🔥

This Markdown version keeps your formatting structured and readable, making it ready for use in blog platforms like Hugo, Jekyll, or GitHub Pages! 🚀


# Add order.

I was not satisfied with this answer. There is no controller implementation. So I add below order to o3-mini high.
```
　Using PID control and Model Predictive Control tor temperature controler.
```

# Answer (2nd)


Below is an extended example that builds on the earlier simulation code. In this version we add two temperature‐control strategies that adjust the boundary temperature in real time:
	1.	PID Control: A classic proportional–integral–derivative controller that, at every time step, measures the (average) temperature error and computes a control signal. This signal is then added to the target temperature to “force” the boundaries hotter (or cooler) so that the overall domain temperature is driven toward the target.
	2.	Model Predictive Control (MPC): A simplified MPC controller that uses a first–order linear model of the average temperature dynamics to “predict” future temperature evolution over a short horizon. It then solves an optimization problem (using a quadratic cost) to pick a sequence of control moves that will drive the predicted temperature toward the target. Only the first control move is applied to update the boundary condition.

In the code below the simulation of the thermal–fluid system (with buoyancy via the Boussinesq approximation) is “wrapped” into a function that at every time step updates the temperature boundary conditions based on the controller’s output. (For simplicity we assume that the control input is added to a base target value to form the Dirichlet condition on every side.) You can choose to run the simulation either with a PID or with an MPC controller.



```python
"""
Thermal–Fluid Simulation with Dynamic Temperature Control
(using both PID control and Model Predictive Control)

This script simulates the thermal–fluid equations in a 2D square domain
with buoyancy (Boussinesq approximation). In addition, it runs a controlled
simulation in which the temperature on all boundaries is adjusted in real time
to drive the average domain temperature toward a desired target using either:
  • A PID controller, or
  • A simplified Model Predictive Controller (MPC)

For MPC we assume a simplified first–order linear model for the evolution of
the average temperature:
    T_avg(k+1) = T_avg(k) + dt*( -a*(T_avg(k) - T0) + b*u[k] )
where u[k] is the control action applied at step k.
"""

import numpy as np
import matplotlib.pyplot as plt
import cvxpy as cp

# ----------------------------------
# Helper Functions: Pressure Solver and BCs
# ----------------------------------
def pressure_poisson(p, dx, dy, b, nit):
    """Solves the pressure Poisson equation using a Jacobi iteration."""
    for it in range(nit):
        pn = p.copy()
        p[1:-1, 1:-1] = (((pn[1:-1, 2:] + pn[1:-1, 0:-2]) * dy**2 +
                          (pn[2:, 1:-1] + pn[0:-2, 1:-1]) * dx**2 -
                          b[1:-1, 1:-1] * dx**2 * dy**2) /
                         (2 * (dx**2 + dy**2)))
        # Homogeneous Neumann conditions on all sides:
        p[:, 0] = p[:, 1]
        p[:, -1] = p[:, -2]
        p[0, :] = p[1, :]
        p[-1, :] = p[-2, :]
    return p

def apply_boundary_conditions(u, v, p, T, bc, dx, dy):
    """
    Applies boundary conditions stored in dictionary `bc` to the fields.
    Each side (left, right, top, bottom) is a sub–dictionary mapping each variable
    to a tuple (condition_type, value) where condition_type is 'dirichlet' or 'neumann'.
    """
    # Left boundary
    if 'left' in bc:
        for var, (cond, val) in bc['left'].items():
            if var == 'u':
                u[:, 0] = val if cond=='dirichlet' else u[:, 1] + val * dx
            elif var == 'v':
                v[:, 0] = val if cond=='dirichlet' else v[:, 1] + val * dx
            elif var == 'T':
                T[:, 0] = val if cond=='dirichlet' else T[:, 1] + val * dx
            elif var == 'p':
                p[:, 0] = val if cond=='dirichlet' else p[:, 1] + val * dx
    # Right boundary
    if 'right' in bc:
        for var, (cond, val) in bc['right'].items():
            if var == 'u':
                u[:, -1] = val if cond=='dirichlet' else u[:, -2] + val * dx
            elif var == 'v':
                v[:, -1] = val if cond=='dirichlet' else v[:, -2] + val * dx
            elif var == 'T':
                T[:, -1] = val if cond=='dirichlet' else T[:, -2] + val * dx
            elif var == 'p':
                p[:, -1] = val if cond=='dirichlet' else p[:, -2] + val * dx
    # Top boundary
    if 'top' in bc:
        for var, (cond, val) in bc['top'].items():
            if var == 'u':
                u[0, :] = val if cond=='dirichlet' else u[1, :] + val * dy
            elif var == 'v':
                v[0, :] = val if cond=='dirichlet' else v[1, :] + val * dy
            elif var == 'T':
                T[0, :] = val if cond=='dirichlet' else T[1, :] + val * dy
            elif var == 'p':
                p[0, :] = val if cond=='dirichlet' else p[1, :] + val * dy
    # Bottom boundary
    if 'bottom' in bc:
        for var, (cond, val) in bc['bottom'].items():
            if var == 'u':
                u[-1, :] = val if cond=='dirichlet' else u[-2, :] + val * dy
            elif var == 'v':
                v[-1, :] = val if cond=='dirichlet' else v[-2, :] + val * dy
            elif var == 'T':
                T[-1, :] = val if cond=='dirichlet' else T[-2, :] + val * dy
            elif var == 'p':
                p[-1, :] = val if cond=='dirichlet' else p[-2, :] + val * dy
    return u, v, p, T

# ----------------------------------
# Plot Function: Temperature Contour
# ----------------------------------
def plot_temperature_contour(x, y, T, title="Temperature Contour"):
    """Plots a contour of the temperature field."""
    X, Y = np.meshgrid(x, y)
    plt.figure(figsize=(6, 5))
    cp_plot = plt.contourf(X, Y, T, 20, cmap="inferno")
    plt.colorbar(cp_plot)
    plt.xlabel("x")
    plt.ylabel("y")
    plt.title(title)
    plt.tight_layout()
    plt.show()

# ----------------------------------
# Controller Classes
# ----------------------------------
class PIDController:
    """
    A simple PID controller.
    
    The update method computes:
         u = Kp * error + Ki * integral + Kd * derivative,
    where error = setpoint - measured_value.
    """
    def __init__(self, Kp, Ki, Kd, setpoint, dt):
        self.Kp = Kp
        self.Ki = Ki
        self.Kd = Kd
        self.setpoint = setpoint
        self.dt = dt
        self.integral = 0.0
        self.previous_error = 0.0

    def update(self, measurement):
        error = self.setpoint - measurement
        self.integral += error * self.dt
        derivative = (error - self.previous_error) / self.dt
        self.previous_error = error
        return self.Kp * error + self.Ki * self.integral + self.Kd * derivative

class MPCController:
    """
    A simplified Model Predictive Controller (MPC) for the average temperature.
    
    We assume a linear model:
         T(k+1) = T(k) + dt*( -a*(T - T0) + b*u )
    over a prediction horizon of N steps. The cost to minimize is:
         J = Σ[(T_pred - T_target)^2 + R * u^2]
    The update() method returns the first control move.
    """
    def __init__(self, N, dt, a, b, T0, R=1.0, umin=-10, umax=10):
        self.N = N
        self.dt = dt
        self.a = a
        self.b = b
        self.T0 = T0
        self.R = R
        self.umin = umin
        self.umax = umax

    def update(self, T_current, T_target):
        N = self.N
        dt = self.dt
        a = self.a
        b = self.b

        # Decision variables: predicted temperatures T[0..N] and control moves u[0..N-1]
        T_pred = cp.Variable(N + 1)
        u_seq = cp.Variable(N)
        cost = 0
        constraints = [T_pred[0] == T_current]

        for i in range(N):
            cost += cp.square(T_pred[i + 1] - T_target) + self.R * cp.square(u_seq[i])
            constraints += [T_pred[i + 1] == T_pred[i] + dt * (-a * (T_pred[i] - self.T0) + b * u_seq[i])]
            constraints += [u_seq[i] >= self.umin, u_seq[i] <= self.umax]

        # Add reasonable bounds on T_pred to avoid unboundedness.
        constraints += [T_pred >= self.T0 - 50, T_pred <= T_target + 50]

        prob = cp.Problem(cp.Minimize(cost), constraints)

        # Try using OSQP; if it fails, try SCS.
        try:
            prob.solve(solver=cp.OSQP, verbose=False)
        except cp.error.SolverError:
            prob.solve(solver=cp.SCS, verbose=False)

        if u_seq.value is not None:
            return u_seq.value[0]
        else:
            return 0.0

# ----------------------------------
# Controlled Simulation Function
# ----------------------------------
def simulate_controlled(controller, T_target=320.0, nx=50, ny=50, nt=500, nit=50, dt=0.001,
                        L=1.0, nu=0.1, alpha=0.01, rho=1.0, g=9.81, beta=0.1, T0=300.0):
    """
    Simulates the thermal–fluid system with a temperature controller.
    
    The controller (either a PIDController or MPCController) is used at each time step
    to compute a control signal based on the current average temperature. That control
    signal is then added to the target temperature to set Dirichlet temperature boundary conditions.
    
    Returns:
       x, y                : spatial grids
       u, v, p, T          : final fields
       flow_rate           : an estimate of the net flow rate
       avg_T_history       : history of the average domain temperature
       control_history     : history of the control signal
    """
    dx = L / (nx - 1)
    dy = L / (ny - 1)
    x = np.linspace(0, L, nx)
    y = np.linspace(0, L, ny)
    
    # Initialize the fields
    u = np.zeros((ny, nx))         # x–velocity
    v = np.zeros((ny, nx))         # y–velocity
    p = np.zeros((ny, nx))         # Pressure
    T = np.ones((ny, nx)) * T0      # Temperature (starts at T0)
    
    # Define initial boundary conditions.
    # The temperature on all boundaries is initially set to the target.
    bc = {
        'left':   {'u': ('dirichlet', 1.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', T_target),
                   'p': ('dirichlet', 0.0)},
        'right':  {'u': ('neumann', 0.0),
                   'v': ('neumann', 0.0),
                   'T': ('dirichlet', T_target),
                   'p': ('dirichlet', 0.0)},
        'top':    {'u': ('dirichlet', 0.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', T_target),
                   'p': ('neumann', 0.0)},
        'bottom': {'u': ('dirichlet', 0.0),
                   'v': ('dirichlet', 0.0),
                   'T': ('dirichlet', T_target),
                   'p': ('neumann', 0.0)}
    }
    
    avg_T_history = []
    control_history = []
    
    # Time stepping loop
    for n in range(nt):
        un = u.copy()
        vn = v.copy()
        Tn = T.copy()
        
        # Compute the source term for pressure Poisson (from continuity)
        b_field = np.zeros((ny, nx))
        b_field[1:-1, 1:-1] = (rho * (1/dt) *
                               ((un[1:-1, 2:] - un[1:-1, 0:-2]) / (2 * dx) +
                                (vn[2:, 1:-1] - vn[0:-2, 1:-1]) / (2 * dy)))
        
        # --- Update the Temperature Field (Energy Equation) ---
        T[1:-1, 1:-1] = (Tn[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (Tn[1:-1, 1:-1] - Tn[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (Tn[1:-1, 1:-1] - Tn[0:-2, 1:-1]) / dy) +
                         dt * alpha * ((Tn[1:-1, 2:] - 2 * Tn[1:-1, 1:-1] + Tn[1:-1, 0:-2]) / dx**2 +
                                       (Tn[2:, 1:-1] - 2 * Tn[1:-1, 1:-1] + Tn[0:-2, 1:-1]) / dy**2))
        
        # --- Update the u–momentum (x–direction) ---
        u[1:-1, 1:-1] = (un[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (un[1:-1, 1:-1] - un[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (un[1:-1, 1:-1] - un[0:-2, 1:-1]) / dy) -
                         dt * (1 / rho) * ((p[1:-1, 2:] - p[1:-1, 0:-2]) / (2 * dx)) +
                         dt * nu * ((un[1:-1, 2:] - 2 * un[1:-1, 1:-1] + un[1:-1, 0:-2]) / dx**2 +
                                    (un[2:, 1:-1] - 2 * un[1:-1, 1:-1] + un[0:-2, 1:-1]) / dy**2))
        
        # --- Update the v–momentum (y–direction) with buoyancy ---
        v[1:-1, 1:-1] = (vn[1:-1, 1:-1] -
                         dt * (un[1:-1, 1:-1] * (vn[1:-1, 1:-1] - vn[1:-1, 0:-2]) / dx +
                               vn[1:-1, 1:-1] * (vn[1:-1, 1:-1] - vn[0:-2, 1:-1]) / dy) -
                         dt * (1 / rho) * ((p[2:, 1:-1] - p[0:-2, 1:-1]) / (2 * dy)) +
                         dt * nu * ((vn[1:-1, 2:] - 2 * vn[1:-1, 1:-1] + vn[1:-1, 0:-2]) / dx**2 +
                                    (vn[2:, 1:-1] - 2 * vn[1:-1, 1:-1] + vn[0:-2, 1:-1]) / dy**2) +
                         dt * g * beta * (Tn[1:-1, 1:-1] - T0))
        
        # --- Solve for Pressure ---
        p = pressure_poisson(p, dx, dy, b_field, nit)
        
        # --- Controller Update ---
        # Compute the average temperature in the domain (could be replaced by sensor readings)
        T_avg = np.mean(T)
        avg_T_history.append(T_avg)
        # Get control input. For MPC we call update(measurement, target), for PID update(measurement)
        if isinstance(controller, MPCController):
            control_input = controller.update(T_avg, T_target)
        else:
            control_input = controller.update(T_avg)
        control_history.append(control_input)
        
        # Update the temperature boundary conditions: 
        # Here the new boundary temperature is set to (T_target + control_input)
        new_T_boundary = T_target + control_input
        bc['left']['T'] = ('dirichlet', new_T_boundary)
        bc['right']['T'] = ('dirichlet', new_T_boundary)
        bc['top']['T'] = ('dirichlet', new_T_boundary)
        bc['bottom']['T'] = ('dirichlet', new_T_boundary)
        
        # --- Enforce Boundary Conditions ---
        u, v, p, T = apply_boundary_conditions(u, v, p, T, bc, dx, dy)
    
    # Compute an estimate of the flow rate (for example, integrate u on the left boundary)
    flow_rate = np.sum(u[:, 0]) * dy
    return x, y, u, v, p, T, flow_rate, avg_T_history, control_history

# ----------------------------------
# Main Routine: Run PID and MPC Control Simulations
# ----------------------------------
if __name__ == "__main__":
    # Simulation and physical parameters
    nx = 50
    ny = 50
    nt = 1000        # number of time steps
    dt = 0.001
    L = 1.0
    T_target = 320.0  # desired temperature (e.g., Kelvin)
    
    # ----- Run PID-Controlled Simulation -----
    # PID tuning parameters (these may need adjustment)
    Kp = 0.5
    Ki = 0.1
    Kd = 0.01
    pid = PIDController(Kp=Kp, Ki=Ki, Kd=Kd, setpoint=T_target, dt=dt)
    
    print("Running PID-controlled simulation...")
    (x, y, u, v, p, T,
     flow_rate_pid, avg_T_history_pid,
     control_history_pid) = simulate_controlled(controller=pid, T_target=T_target,
                                                 nx=nx, ny=ny, nt=nt, dt=dt, L=L)
    print("PID Control Flow Rate =", flow_rate_pid)
    
    # ----- Run MPC-Controlled Simulation -----
    # MPC model parameters:
    # We assume a simple linear model: T(k+1) = T(k) + dt*( -a*(T - T0) + b*u )
    a = 0.1   # dynamics parameter (e.g., representing cooling rate)
    b = 0.5   # gain representing the effect of control input on temperature
    N = 10    # prediction horizon
    R = 0.01  # weight on control effort
    mpc = MPCController(N=N, dt=dt, a=a, b=b, T0=300.0, R=R, umin=-10, umax=10)
    
    print("Running MPC-controlled simulation...")
    (x, y, u, v, p, T,
     flow_rate_mpc, avg_T_history_mpc,
     control_history_mpc) = simulate_controlled(controller=mpc, T_target=T_target,
                                                 nx=nx, ny=ny, nt=nt, dt=dt, L=L)
    print("MPC Control Flow Rate =", flow_rate_mpc)
    
    # ----- Plot Results: Average Temperature and Control Actions over Time -----
    plt.figure(figsize=(12, 5))
    
    plt.subplot(1, 2, 1)
    plt.plot(avg_T_history_pid, label="PID")
    plt.plot(avg_T_history_mpc, label="MPC")
    plt.axhline(T_target, color='k', linestyle='--', label="Target")
    plt.xlabel("Time Step")
    plt.ylabel("Average Temperature")
    plt.title("Average Temperature History")
    plt.legend()
    
    plt.subplot(1, 2, 2)
    plt.plot(control_history_pid, label="PID Control Input")
    plt.plot(control_history_mpc, label="MPC Control Input")
    plt.xlabel("Time Step")
    plt.ylabel("Control Action")
    plt.title("Control Signal History")
    plt.legend()
    
    plt.tight_layout()
    plt.show()
    
    # ----- Plot Temperature Contour for PID and MPC Results -----
    plot_temperature_contour(x, y, T, title="Final Temperature Contour (MPC)")
    # Note: For demonstration, we plot the last computed T field.
    # In practice you might store T from each simulation separately.
```


Explanation
1.	Simulation of the Thermal–Fluid System:
The functions pressure_poisson() and apply_boundary_conditions() are as before. The main simulation loop (inside simulate_controlled()) advances the velocity, pressure, and temperature fields using finite–difference approximations. In the v–momentum update the buoyancy term

```python
dt \cdot g \cdot \beta (T - T_0)
```
is added.

2.	Dynamic Control Update:
At each time step the average domain temperature is computed (which may represent sensor feedback). The controller’s update() method is then called to compute a control input. This value is added to the base target temperature to set the Dirichlet boundary conditions for the temperature field. In effect, if the average temperature is too low, the control signal will be positive so that the boundaries are “heated” above the target, driving up the internal temperature.

3.	PID vs. MPC:
– The PIDController class holds an internal integral and previous error, and computes a simple control law.
– The MPCController class formulates and solves a short–horizon quadratic optimization (using cvxpy) based on a linear model of the temperature dynamics. Only the first control move is applied.

4.	Running the Simulation:
In the __main__ section the code first runs the simulation using a PID controller and then using the MPC controller. It then plots the history of the average temperature and the control signal over time for comparison.

This example serves as a template. In practice you would need to tune the controller parameters (and possibly the underlying model for MPC) for your application. Enjoy experimenting with your thermal–fluid temperature control simulation!