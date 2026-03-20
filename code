import openseespy.opensees as ops
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import MaxNLocator
import random    
import pandas as pd
from mpl_toolkits.mplot3d import Axes3D
import time
import sys
import os
import csv
import shutil

# =================================================================================
#  GLOBAL: LIST OF EARTHQUAKE FILES (USER PROVIDED)
# =================================================================================
EQ_FILES = [
    r"E:\COLLEGE SVNIT\semester\sem7\FYP\NEARFEILD1.txt",
    r"E:\COLLEGE SVNIT\semester\sem7\FYP\elcentro1.txt"
]

# =================================================================================
# PART 1: CONFIGURATION & INPUTS
# =================================================================================

def get_common_inputs():
    """
    Asks the user for all structural and material parameters ONCE.
    """
    print("\n" + "="*80)
    print("   ADVANCED SEISMIC TMD OPTIMIZATION (LIVE PROGRESS MODE)")
    print("   (Shows Generation Progress, Single Pass Speed, Beast Optimization)")
    print("="*80 + "\n")

    # --- Robust Input Helper ---
    def safe_input(prompt, default):
        try:
            val = input(f"{prompt} [{default}]: ").strip()
            if not val:
                return str(default)
            return val
        except EOFError:
            print(f"{prompt} [{default}]: <EOF detected, using default>")
            return str(default)

    def input_val(prompt, default, type_conv=float):
        val_str = safe_input(prompt, default)
        try:
            return type_conv(val_str)
        except:
            print(f"Invalid input, using default: {default}")
            return default

    config = {}

    # --- 1. Geometry ---
    print("--- 1. BUILDING GEOMETRY ---")
    b_type = safe_input("Building Plan Type (Regular/Irregular)", "Irregular").lower()
    config["building_type"] = "Regular" if b_type == "regular" else "Irregular"
    
    config["num_bays_x"] = input_val("Number of Bays in X-Direction", 3, int)
    config["bay_width_x"] = input_val("Bay Width in X-Direction (m)", 5.0)
    config["num_bays_z"] = input_val("Number of Bays in Z-Direction", 3, int)
    config["bay_width_z"] = input_val("Bay Width in Z-Direction (m)", 4.0)
    
    if config["building_type"] == "Irregular":
        print("   > Irregularity Definition (Re-entrant Corner):")
        config["cutout_x"] = input_val(f"     Nodes removed after Bay X index", 2, int)
        config["cutout_z"] = input_val(f"     Nodes removed after Bay Z index", 2, int)
    else:
        config["cutout_x"] = 999
        config["cutout_z"] = 999
    
    config["story_height"] = input_val("Typical Story Height (m)", 3.2)
    floors_str = safe_input("Floors to Simulate (comma-separated, e.g. 5,10)", "5")
    config["floors_list"] = [int(x.strip()) for x in floors_str.split(',')]

    # --- 2. TMD Setup ---
    print("\n--- 2. TMD CONFIGURATION ---")
    config["manual_tmd"] = False
    config["tmd_coords"] = []
    
    num_tmds = 2 if config["building_type"] == "Irregular" else 1
    print(f"   > Based on geometry, {num_tmds} TMD(s) will be optimized.")
    
    manual_loc = safe_input("     Manually specify TMD coordinates? (y/n)", "n").lower()
    if manual_loc == 'y':
        config["manual_tmd"] = True
        print("     Enter coordinates (X, Z) for TMD placement:")
        for i in range(num_tmds):
            mx = input_val(f"       TMD {i+1} X Coordinate (m)", 0.0)
            mz = input_val(f"       TMD {i+1} Z Coordinate (m)", 0.0)
            config["tmd_coords"].append((mx, mz))

    # --- 3. Material & Loads ---
    print("\n--- 3. MATERIAL & LOADS (IS 456:2000) ---")
    config["fck"] = input_val("Concrete Grade fck (MPa)", 30, float)
    config["fy"] = input_val("Steel Grade fy (MPa)", 415, float)
    # IS 456:2000 Cl 6.2.3.1: E = 5000 sqrt(fck)
    config["E_conc"] = 5000 * np.sqrt(config["fck"]) * 1000 
    
    config["finish_thick"] = input_val("Floor Finish Thickness (m)", 0.05)
    config["live_load"] = input_val("Live Load (kN/m2)", 3.0)

    # --- 4. Seismic Settings ---
    print("\n--- 4. SEISMIC PARAMETERS (IS 1893) ---")
    zone_map = {'II': 0.10, 'III': 0.16, 'IV': 0.24, 'V': 0.36}
    zone_in = safe_input("Seismic Zone (II, III, IV, V)", "V").upper()
    config["zone_factor"] = zone_map.get(zone_in, 0.36)
    config["damping_ratio"] = input_val("Structure Damping Ratio (zeta)", 0.02)
    
    # --- 5. Analysis Settings ---
    print("\n--- 5. ANALYSIS SETTINGS ---")
    config["dt_target"] = input_val("Analysis Time Step (dt)", 0.02)
    config["gm_unit_g"] = True 
    config["scale_factor"] = input_val("GM Scale Factor", 1.0)
    config["duration"] = input_val("Max Duration to Analyze (s)", 30.0) 

    # --- 6. Optimization ---
    print("\n--- 6. GA OPTIMIZATION SETTINGS ---")
    # AGGRESSIVE SEARCH SETTINGS (BEAST MODE)
    config["pop_size"] = input_val("Population Size", 15, int)      
    config["generations"] = input_val("Generations", 8, int)       
    
    config["mu_min"] = input_val("Mass Ratio Min", 0.02) # Min 2%
    config["mu_max"] = input_val("Mass Ratio Max", 0.15) # Max 15% (Heavy TMD for large reduction)
    config["ga_seed"] = 42

    config["save_folder"] = "Results_Final_Project"
    
    return config

# =================================================================================
# PART 2: PRELIMINARY DESIGN
# =================================================================================

class PreliminaryDesign:
    def __init__(self, config, num_stories, stiffness_scale=1.0):
        self.cfg = config
        self.num_stories = num_stories
        self.stiffness_scale = stiffness_scale
        self.perform_sizing()

    def perform_sizing(self):
        max_span = max(self.cfg["bay_width_x"], self.cfg["bay_width_z"])
        
        # 1. Beam Sizing (Span/12 rule)
        depth_est = (max_span * 1000) / 12 
        self.beam_depth = round(depth_est / 50) * 50 / 1000 
        self.beam_width = max(0.23, round((self.beam_depth * 0.5) / 0.05) * 0.05)
        
        # 2. Column Sizing
        # Base size increased based on stories + scaling factor
        initial_size = 0.30
        if self.cfg["zone_factor"] > 0.3: initial_size = 0.40
        
        base_dim = initial_size + max(0, (self.num_stories - 3) * 0.05)
        
        # Apply Stiffness Scaling (Critical for Drift Compliance Loop)
        base_dim = base_dim * self.stiffness_scale
        
        # Round to nearest 50mm
        self.col_dim = round(base_dim / 0.05) * 0.05 
        
        # Cap to realistic maximum
        if self.col_dim > 1.5:
             print(f"   [Warning] Calculated Column Size {self.col_dim:.2f}m is very large. Capping at 1.5m.")
             self.col_dim = 1.5

        # 3. Mass/Load Calculation
        slab_thick = 0.150
        sw_slab = slab_thick * 25.0 
        ff_load = self.cfg["finish_thick"] * 24.0 
        # IS 1893 Table 8: 25% LL for LL <= 3.0, else 50%
        percent_ll = 0.25 if self.cfg["live_load"] <= 3.0 else 0.50
        eff_ll = self.cfg["live_load"] * percent_ll
        self.floor_mass_density = (sw_slab + ff_load + eff_ll) 

    def save_design_report(self, folder):
        path = os.path.join(folder, "Design_Report.txt")
        with open(path, "w") as f:
            f.write(f"DESIGN REPORT (IS 456 / IS 1893)\n")
            f.write(f"----------------------------------------\n")
            f.write(f"Stories: {self.num_stories}\n")
            f.write(f"Stiffness Scale Factor applied: {self.stiffness_scale:.2f}\n")
            f.write(f"Column Size: {self.col_dim*1000:.0f} mm x {self.col_dim*1000:.0f} mm\n")
            f.write(f"Beam Size: {self.beam_width*1000:.0f} mm x {self.beam_depth*1000:.0f} mm\n")
            f.write(f"Floor Seismic Mass: {self.floor_mass_density:.2f} kN/m2\n")

# =================================================================================
# PART 3: OPENSEES MODEL
# =================================================================================

class StructureModel:
    def __init__(self, design, config, num_stories):
        self.design = design
        self.cfg = config
        self.num_stories = num_stories
        self.node_map = {} 
        self.roof_nodes = []
        self.valid_coords = [] 

    def build_model(self, tmd_params=None, export_K=False, folder=None):
        ops.wipe()
        ops.model('basic', '-ndm', 3, '-ndf', 6)
        
        nx, ny, nz = self.cfg["num_bays_x"]+1, self.num_stories+1, self.cfg["num_bays_z"]+1
        
        self.valid_coords = []
        for i in range(nx):
            for k in range(nz):
                if self.cfg["building_type"] == "Irregular":
                    # Simple L-shape logic
                    if i > self.cfg["cutout_x"] and k > self.cfg["cutout_z"]:
                        continue 
                self.valid_coords.append((i, k))

        node_tag = 1
        self.roof_nodes = []
        self.node_map = {}

        for j in range(ny):
            y = j * self.cfg["story_height"]
            for i, k in self.valid_coords:
                x, z = i * self.cfg["bay_width_x"], k * self.cfg["bay_width_z"]
                ops.node(node_tag, x, y, z)
                self.node_map[(i,j,k)] = node_tag
                
                if j == 0:
                    ops.fix(node_tag, 1, 1, 1, 1, 1, 1)
                else:
                    trib_area = self.cfg["bay_width_x"] * self.cfg["bay_width_z"]
                    # Mass = Weight / g (Tonnes)
                    m_val = (self.design.floor_mass_density * trib_area) / 9.81 
                    ops.mass(node_tag, m_val, m_val, m_val, 1e-6, 1e-6, 1e-6)
                    if j == ny - 1: self.roof_nodes.append(node_tag)
                node_tag += 1

        ops.geomTransf('Linear', 1, 0, 0, 1) # Columns
        ops.geomTransf('Linear', 2, 0, 1, 0) # Beams X
        ops.geomTransf('Linear', 3, 1, 0, 0) # Beams Z
        
        E, G = self.cfg["E_conc"], self.cfg["E_conc"]/(2.4)
        
        Ac = self.design.col_dim**2
        Ic = 0.70 * (self.design.col_dim**4)/12 
        
        Ab = self.design.beam_width * self.design.beam_depth
        Iby = 0.35 * (self.design.beam_width * self.design.beam_depth**3)/12 
        Ibz = 0.35 * (self.design.beam_depth * self.design.beam_width**3)/12
        J = (Iby + Ibz) 
        
        elem_tag = 1
        for j in range(ny - 1): # Columns
            for i, k in self.valid_coords:
                n1, n2 = self.node_map[(i,j,k)], self.node_map[(i,j+1,k)]
                ops.element('elasticBeamColumn', elem_tag, n1, n2, Ac, E, G, 2*Ic, Ic, Ic, 1)
                elem_tag += 1
        
        for j in range(1, ny): # Beams
            for i, k in self.valid_coords:
                if (i+1, k) in self.valid_coords: # X Beam
                    n1, n2 = self.node_map[(i,j,k)], self.node_map[(i+1,j,k)]
                    ops.element('elasticBeamColumn', elem_tag, n1, n2, Ab, E, G, J, Iby, Ibz, 2)
                    elem_tag += 1
                if (i, k+1) in self.valid_coords: # Z Beam
                    n1, n2 = self.node_map[(i,j,k)], self.node_map[(i,j,k+1)]
                    ops.element('elasticBeamColumn', elem_tag, n1, n2, Ab, E, G, J, Iby, Ibz, 3)
                    elem_tag += 1

        if export_K and folder:
            self.export_stiffness_matrix(folder)

        if tmd_params:
            self.apply_tmds(tmd_params, node_tag, elem_tag)

    def export_stiffness_matrix(self, folder):
        ops.system('FullGeneral')
        ops.analysis('Transient')
        ops.integrator('GimmeMCK', 0.0, 0.0, 1.0)
        ops.analyze(1, 0.0)
        path = os.path.join(folder, 'Global_K.out')
        ops.printA('-file', path)

    def apply_tmds(self, params_list, start_node, start_elem):
        t_node, t_mat, t_ele = start_node, 1000, start_elem
        for tmd in params_list:
            loc_node = tmd['node']
            loc = ops.nodeCoord(loc_node)
            ops.node(t_node, loc[0], loc[1], loc[2])
            ops.fix(t_node, 0, 1, 1, 1, 1, 1) # Slide X only
            
            m = tmd['mass']
            ops.mass(t_node, m, 1e-10, 1e-10, 1e-10, 1e-10, 1e-10)
            
            ops.uniaxialMaterial('Elastic', t_mat, tmd['k'])
            ops.uniaxialMaterial('Viscous', t_mat+1, tmd['c'], 1)
            ops.uniaxialMaterial('Parallel', t_mat+2, t_mat, t_mat+1)
            
            ops.element('twoNodeLink', t_ele, loc_node, t_node, '-mat', t_mat+2, '-dir', 1, '-doRayleigh', 0)
            t_node += 1; t_mat += 5; t_ele += 1

    def get_nearest_node(self, x, z):
        min_dist = float('inf')
        nearest = self.roof_nodes[0]
        for node in self.roof_nodes:
            coords = ops.nodeCoord(node)
            dist = np.sqrt((coords[0] - x)**2 + (coords[2] - z)**2)
            if dist < min_dist:
                min_dist = dist
                nearest = node
        return nearest

    def get_tmd_locations(self):
        self.build_model()
        if self.cfg["manual_tmd"]:
            nodes = []
            for coords in self.cfg["tmd_coords"]:
                node = self.get_nearest_node(coords[0], coords[1])
                nodes.append(node)
            return nodes
        else:
            try:
                ops.eigen(1)
                roof_disps = []
                for n in self.roof_nodes:
                    val = ops.nodeEigenvector(n, 1, 1) 
                    roof_disps.append((n, abs(val)))
                roof_disps.sort(key=lambda x: x[1], reverse=True)
                num_tmds = 2 if self.cfg["building_type"] == "Irregular" else 1
                best_nodes = [x[0] for x in roof_disps[:num_tmds]]
                return best_nodes
            except:
                return [self.roof_nodes[0]]
    
    def get_drift_check_nodes(self):
        valid_col_coords = self.valid_coords[0] 
        i, k = valid_col_coords
        col_nodes = []
        for j in range(self.num_stories + 1): 
            if (i, j, k) in self.node_map:
                col_nodes.append(self.node_map[(i,j,k)])
        return col_nodes

# =================================================================================
# PART 4: ANALYSIS
# =================================================================================

def load_ground_motion(cfg):
    path = cfg["gm_file"]
    
    if not os.path.exists(path):
        if os.path.exists(path + ".txt"): path += ".txt"
        elif os.path.exists(path + ".dat"): path += ".dat"
        else: 
            print(f"   [Error] File not found: {path}. Using synthetic.")
            return generate_synthetic(cfg)
    
    print(f"   > Loading Ground Motion: {os.path.basename(path)}")
    try:
        raw_acc = np.loadtxt(path)
        if len(raw_acc.shape) > 1: raw_acc = raw_acc[:, 0]
    except:
        return generate_synthetic(cfg)
            
    dt_orig = 0.02
    dt_targ = cfg["dt_target"]
    
    # Unit Handling
    peak_val = np.max(np.abs(raw_acc))
    if peak_val > 20.0:
        print(f"     [Unit Check] Peak={peak_val:.2f}. Assuming cm/s^2. Converting to m/s^2.")
        raw_acc = raw_acc / 100.0 
    elif peak_val > 2.0:
        print(f"     [Unit Check] Peak={peak_val:.2f}. Assuming m/s^2.")
        pass 
    else:
        print(f"     [Unit Check] Peak={peak_val:.2f}. Assuming 'g'. Multiplying by 9.81.")
        raw_acc = raw_acc * 9.81
        
    raw_acc = raw_acc * cfg["scale_factor"]
    
    n_orig = len(raw_acc)
    # CLIP DURATION TO 30s
    file_duration = n_orig * dt_orig
    target_duration = min(file_duration, cfg["duration"]) 
    
    t_orig = np.linspace(0, file_duration, n_orig)
    n_targ = int(target_duration / dt_targ)
    t_targ = np.linspace(0, target_duration, n_targ)
    
    acc_resampled = np.interp(t_targ, t_orig, raw_acc)
    
    return t_targ, acc_resampled

def generate_synthetic(cfg):
    print("   > Generating Synthetic White-Noise GM...")
    dt = cfg["dt_target"]
    dur = cfg["duration"]
    t = np.arange(0, dur, dt)
    noise = np.random.normal(0, 1, len(t))
    envelope = np.exp(-0.5 * (t - 5.0)**2)
    acc = noise * envelope
    pga = cfg["zone_factor"] * 9.81
    acc = acc / np.max(np.abs(acc)) * pga
    return t, acc

def run_time_history(model, acc, dt, record_nodes, damping_ratio, fixed_w1=None):
    """
    Runs time history analysis.
    fixed_w1: If provided, uses this fundamental frequency to calculate beta_k.
              If None, calculates from current model eigenvalues.
    """
    
    # --- DAMPING CALCULATION ---
    if fixed_w1 is not None:
        w1 = fixed_w1
    else:
        # Calculate w1 from current (bare) model
        vals = ops.eigen(1)
        w1 = np.sqrt(vals[0])

    beta_k = (2 * damping_ratio) / w1
    
    # Apply Rayleigh Damping (Stiffness Proportional)
    # Note: TMD Links have -doRayleigh 0, so they won't pick this up.
    ops.rayleigh(0.0, 0.0, 0.0, beta_k)
    
    # --- ANALYSIS SETUP ---
    ops.timeSeries('Path', 1, '-dt', dt, '-values', *acc.tolist(), '-factor', 1.0)
    ops.pattern('UniformExcitation', 100, 1, '-accel', 1)
    
    ops.constraints('Transformation')
    ops.numberer('RCM')
    ops.system('BandGeneral')
    ops.test('NormDispIncr', 1.0e-4, 10)
    
    ops.algorithm('Newton')
    ops.integrator('Newmark', 0.5, 0.25)
    
    ops.analysis('Transient')
    
    N = len(acc)
    results = {'time': [], 'disp': [], 'max_drift_profile': []}
    monitor = record_nodes[0]
    
    col_nodes = model.get_drift_check_nodes() 
    num_stories = model.num_stories
    story_height = model.cfg["story_height"]
    max_drifts = np.zeros(num_stories)
    
    for i in range(N):
        ops.analyze(1, dt)
        results['time'].append(ops.getTime())
        results['disp'].append(ops.nodeDisp(monitor, 1))
        
        for s in range(num_stories):
            node_top = col_nodes[s+1]
            node_bot = col_nodes[s]
            drift_ratio = abs(ops.nodeDisp(node_top, 1) - ops.nodeDisp(node_bot, 1)) / story_height
            if drift_ratio > max_drifts[s]: 
                max_drifts[s] = drift_ratio
    
    results['max_drift_profile'] = max_drifts
    return results, w1

# =================================================================================
# PART 5: GENETIC ALGORITHM
# =================================================================================

class GA_Optimizer:
    def __init__(self, model, cfg, quake_data, target_nodes, w1_bare, uncontrolled_drift):
        self.model = model
        self.cfg = cfg
        self.t, self.acc = quake_data
        self.target_nodes = target_nodes
        self.num_tmds = len(target_nodes)
        self.w1_bare = w1_bare  # Fixed bare structure frequency
        self.base_drift = uncontrolled_drift
        random.seed(cfg["ga_seed"])
        
    def run(self):
        pop_size = self.cfg["pop_size"]
        pop = []
        for _ in range(pop_size):
            gene = []
            for _ in range(self.num_tmds):
                # Init with wider range to explore:
                # Gene format: [mu, fr, zeta]
                # mu: Mass Ratio (0.02 - 0.15)
                # fr: Frequency Ratio (0.70 - 1.30)
                # zeta: Damping Ratio (0.01 - 0.20)
                gene.extend([
                    random.uniform(self.cfg["mu_min"], self.cfg["mu_max"]), 
                    random.uniform(0.70, 1.30),
                    random.uniform(0.01, 0.20)
                ])
            pop.append(gene)
            
        best_gene = None
        convergence = []
        
        # Calculate total mass for ratio conversion
        self.model.build_model()
        vals = ops.eigen(1)
        w_struct = np.sqrt(vals[0])
        total_mass = sum([ops.nodeMass(n, 1) for n in self.model.node_map.values() if n not in range(1, len(self.model.valid_coords)+1)])
        
        for gen in range(self.cfg["generations"]):
            scores = []
            for gene in pop:
                tmd_params = []
                total_mu = 0
                for i in range(self.num_tmds):
                    # Extract 3 parameters per TMD
                    idx = i * 3
                    mu = gene[idx]
                    fr = gene[idx+1]
                    zeta = gene[idx+2]
                    
                    total_mu += mu
                    m_tmd = mu * total_mass 
                    w_tmd = fr * w_struct # Tune relative to bare structure
                    k_tmd = m_tmd * w_tmd**2
                    c_tmd = 2 * zeta * m_tmd * w_tmd
                    
                    tmd_params.append({'mass': m_tmd, 'k': k_tmd, 'c': c_tmd, 'node': self.target_nodes[i]})
                
                self.model.build_model(tmd_params)
                # Pass fixed w1 to ensure consistent damping
                res, _ = run_time_history(self.model, self.acc, self.cfg["dt_target"], self.target_nodes, self.cfg["damping_ratio"], fixed_w1=self.w1_bare)
                
                # --- OBJECTIVE FUNCTION: MINIMIZE DRIFT ---
                max_drift = np.max(res['max_drift_profile'])
                
                # STRICT PENALTY: FITNESS IS ZERO IF DRIFT IS WORSE
                if max_drift >= self.base_drift:
                    fitness = 1e-10 
                else:
                    # Higher fitness for lower drift
                    fitness = 1.0 / max_drift 
                
                scores.append((fitness, gene, max_drift))
            
            # Sort by fitness (descending)
            scores.sort(key=lambda x: x[0], reverse=True)
            best_gene = scores[0][1]
            convergence.append(scores[0][2]) # Storing best drift of generation
            
            # Print Progress
            print(f"      > [GA Progress] Generation {gen+1}/{self.cfg['generations']} - Best Drift: {scores[0][2]:.5f}")

            # Tournament Selection
            new_pop = []
            # Elitism: Keep best 2
            new_pop.append(scores[0][1])
            new_pop.append(scores[1][1])
            
            while len(new_pop) < pop_size:
                # Tournament 1
                candidates1 = random.sample(scores, 3)
                candidates1.sort(key=lambda x: x[0], reverse=True)
                p1 = candidates1[0][1]
                
                # Tournament 2
                candidates2 = random.sample(scores, 3)
                candidates2.sort(key=lambda x: x[0], reverse=True)
                p2 = candidates2[0][1]
                
                child = []
                for i in range(len(p1)):
                    # Crossover (Uniform)
                    val = p1[i] if random.random() > 0.5 else p2[i]
                    
                    # Mutation (Gaussian-ish)
                    if random.random() < 0.20: 
                        val *= random.uniform(0.9, 1.1)
                        # Clamping
                        param_idx = i % 3
                        if param_idx == 0: # Mass Ratio
                            val = max(self.cfg["mu_min"], min(self.cfg["mu_max"], val))
                        elif param_idx == 1: # Freq Ratio
                            val = max(0.7, min(1.3, val))
                        else: # Damping Ratio
                            val = max(0.01, min(0.30, val))
                            
                    child.append(val)
                new_pop.append(child)
            pop = new_pop
            
        return best_gene, convergence, total_mass

# =================================================================================
# PART 6: MAIN EXECUTION
# =================================================================================

if __name__ == "__main__":
    
    common_cfg = get_common_inputs()
    
    if os.path.exists(common_cfg["save_folder"]):
        shutil.rmtree(common_cfg["save_folder"])
    os.makedirs(common_cfg["save_folder"])
    
    comparison_results = []
    
    for gm_path in EQ_FILES:
        eq_name = os.path.basename(gm_path)
        if "." in eq_name: eq_name = eq_name.split('.')[0]
        
        # Determine Field Type based on Filename
        field_type = "Near Field" if "near" in eq_name.lower() else "Far Field"
        
        cfg = common_cfg.copy()
        cfg["gm_file"] = gm_path
        
        eq_folder = os.path.join(cfg["save_folder"], f"EQ_{eq_name}")
        os.makedirs(eq_folder)
        
        print("\n" + "="*80)
        print(f" STARTING ANALYSIS FOR: {eq_name} ({field_type})")
        print("="*80)

        t_arr, acc_arr = load_ground_motion(cfg)
        
        for n_stories in cfg["floors_list"]:
            print(f"\n   >>> PROCESSING {n_stories}-STORY BUILDING...")
            
            # --- SINGLE PASS AGGRESSIVE OPTIMIZATION ---
            design = PreliminaryDesign(cfg, n_stories)
            model = StructureModel(design, cfg, n_stories)
            tmd_nodes = model.get_tmd_locations()
            model.build_model()
            
            # 2. UNCONTROLLED ANALYSIS
            print(f"     -> Running Uncontrolled Analysis...")
            base_res, w1_bare = run_time_history(model, acc_arr, cfg["dt_target"], tmd_nodes, cfg["damping_ratio"])
            drift_base_val = np.max(base_res['max_drift_profile'])
            
            print(f"     -> [DEBUG] Bare Structure T1 = {2*np.pi/w1_bare:.3f} s")
            print(f"     -> Uncontrolled Drift: {drift_base_val:.5f}. Running BEAST MODE GA...")

            # 3. GENETIC ALGORITHM
            # Pass drift baseline to optimizer for penalty check
            optimizer = GA_Optimizer(model, cfg, (t_arr, acc_arr), tmd_nodes, w1_bare, drift_base_val)
            best_gene, hist, total_mass = optimizer.run()
            
            print(f"     -> [DEBUG] Building Mass: {total_mass:.2f} Tonnes")

            # 4. CONTROLLED ANALYSIS
            tmd_final = []
            mass_ratios_list = []
            damping_ratios_list = []
            
            for i in range(len(tmd_nodes)):
                idx = i * 3
                mu = best_gene[idx]
                fr = best_gene[idx+1]
                zeta = best_gene[idx+2]
                
                mass_ratios_list.append(mu)
                damping_ratios_list.append(zeta)
                
                m = mu * total_mass
                w = fr * w1_bare 
                k = m * w**2
                c = 2 * zeta * m * w
                
                tmd_final.append({'mass': m, 'k': k, 'c': c, 'node': tmd_nodes[i]})
            
            mass_ratios_str = ", ".join([f"{m:.4f}" for m in mass_ratios_list])
            
            model.build_model(tmd_final)
            # Use fixed_w1=w1_bare for consistent damping comparison
            opt_res, _ = run_time_history(model, acc_arr, cfg["dt_target"], tmd_nodes, cfg["damping_ratio"], fixed_w1=w1_bare)
            drift_tmd = np.max(opt_res['max_drift_profile'])
            
            print(f"     -> Controlled Drift: {drift_tmd:.5f}")

            # 5. GENERATE OUTPUTS
            print("     -> Generating Output Files & Graphs...")
            
            # Stats for Reporting
            drift_no_tmd = np.max(base_res['max_drift_profile'])
            drift_tmd_val = np.max(opt_res['max_drift_profile'])
            disp_no_tmd = np.max(np.abs(base_res['disp']))
            disp_tmd_val = np.max(np.abs(opt_res['disp']))
            
            red_drift = (drift_no_tmd - drift_tmd_val) / drift_no_tmd * 100
            red_disp = (disp_no_tmd - disp_tmd_val) / disp_no_tmd * 100
            
            comparison_results.append({
                'EQ': eq_name,
                'Stories': n_stories,
                'Drift_Base': drift_no_tmd,
                'Drift_TMD': drift_tmd_val,
                'Disp_Base': disp_no_tmd,
                'Disp_TMD': disp_tmd_val,
                'Reduction': red_drift
            })

            # --- FILE 1: FINAL REDUCTION REPORT ---
            report_path = os.path.join(eq_folder, "Final_Reduction_Report.txt")
            with open(report_path, "w") as f:
                f.write(f"ANALYSIS SUMMARY: {eq_name} ({field_type})\n")
                f.write(f"Stories: {n_stories}\n")
                f.write("="*60 + "\n\n")
                
                f.write("1. TMD LOCATIONS & PARAMETERS\n")
                f.write("-" * 30 + "\n")
                for i, tmd in enumerate(tmd_final):
                    node = tmd['node']
                    loc = ops.nodeCoord(node)
                    f.write(f"   TMD {i+1}:\n")
                    f.write(f"     Node ID: {node}\n")
                    f.write(f"     Coords : (X={loc[0]:.2f}m, Z={loc[2]:.2f}m)\n")
                    f.write(f"     Mass Ratio (mu): {mass_ratios_list[i]:.4f}\n")
                    f.write(f"     Damping Ratio (zeta): {damping_ratios_list[i]:.4f}\n")
                    f.write(f"     Mass (kg): {tmd['mass']:.2f}\n\n")
                
                f.write("2. RESPONSE REDUCTION STATS\n")
                f.write("-" * 30 + "\n")
                f.write(f"   Max Roof Displacement (m):\n")
                f.write(f"     Without TMD: {disp_no_tmd:.5f}\n")
                f.write(f"     With TMD   : {disp_tmd_val:.5f}\n")
                f.write(f"     Reduction  : {red_disp:.2f} %\n\n")
                
                f.write(f"   Max Inter-Story Drift:\n")
                f.write(f"     Without TMD: {drift_no_tmd:.5f}\n")
                f.write(f"     With TMD   : {drift_tmd_val:.5f}\n")
                f.write(f"     Reduction  : {red_drift:.2f} %\n")
            
            # Save raw files for future use
            shutil.copy(report_path, os.path.join(eq_folder, "TMD_Location_and_Stats.txt"))

            # CSVs
            design.save_design_report(eq_folder)
            model.build_model(export_K=True, folder=eq_folder)
            
            min_len = min(len(t_arr), len(base_res['disp']), len(opt_res['disp']))
            df_resp = pd.DataFrame({
                'Time': t_arr[:min_len],
                'Disp_NoTMD_m': base_res['disp'][:min_len],
                'Disp_WithTMD_m': opt_res['disp'][:min_len]
            })
            df_resp.to_csv(os.path.join(eq_folder, "Response_THR.csv"), index=False)
            
            df_drift = pd.DataFrame({
                'Story': np.arange(1, n_stories + 1),
                'Drift_NoTMD': base_res['max_drift_profile'],
                'Drift_WithTMD': opt_res['max_drift_profile']
            })
            df_drift.to_csv(os.path.join(eq_folder, "Drift_Profile.csv"), index=False)
            
            # --- GRAPH 1: ROOF DISPLACEMENT TIME HISTORY ---
            plt.figure(figsize=(10, 6))
            plt.plot(base_res['time'], np.array(base_res['disp'])*1000, 'r--', alpha=0.6, label='Without TMD')
            plt.plot(opt_res['time'], np.array(opt_res['disp'])*1000, 'b-', label='With TMD')
            
            title_text = f"Roof Displacement: {eq_name} ({field_type})"
            plt.title(title_text, fontsize=12, fontweight='bold')
            plt.xlabel("Time (s)")
            plt.ylabel("Roof Displacement (mm)")
            plt.legend(loc='upper right')
            plt.grid(True, alpha=0.3)
            
            # Annotate Optimum Mass Ratio
            info_text = f"Optimum $\mu$: {mass_ratios_str}\nReduction: {red_disp:.1f}%"
            plt.annotate(info_text, xy=(0.02, 0.95), xycoords='axes fraction', 
                         bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="black", alpha=0.8))
            
            plt.savefig(os.path.join(eq_folder, f"Response_TimeHistory_{eq_name}.png"))
            plt.close()

            # --- GRAPH 2: DRIFT PROFILE (NO LIMIT LINE + INTEGER Y-AXIS) ---
            fig_drift, ax_drift = plt.subplots(figsize=(7, 9))
            story_levels = np.arange(1, n_stories + 1)
            ax_drift.plot(base_res['max_drift_profile'], story_levels, 'r--o', label='Without TMD')
            ax_drift.plot(opt_res['max_drift_profile'], story_levels, 'b-o', label='With TMD')
            
            ax_drift.set_title(f"Inter-Story Drift: {eq_name} ({field_type})", fontsize=12, fontweight='bold')
            ax_drift.set_xlabel("Drift Ratio")
            ax_drift.set_ylabel("Story Level")
            ax_drift.yaxis.set_major_locator(MaxNLocator(integer=True)) # Force integer ticks
            ax_drift.grid(True, alpha=0.3)
            ax_drift.legend(loc='best')
            
            # Annotate Optimum Mass Ratio
            ax_drift.annotate(info_text, xy=(0.02, 0.95), xycoords='axes fraction', 
                         bbox=dict(boxstyle="round,pad=0.3", fc="white", ec="black", alpha=0.8))

            plt.savefig(os.path.join(eq_folder, f"Response_DriftProfile_{eq_name}.png"))
            plt.close()

            # --- GRAPH 3: GA CONVERGENCE ---
            if hist:
                plt.figure(figsize=(8, 5))
                plt.plot(range(1, len(hist)+1), hist, 'g-o')
                plt.title(f"GA Convergence: {eq_name} ({field_type})", fontsize=12)
                plt.xlabel("Generation")
                plt.ylabel("Best Max Drift (Ratio)")
                plt.grid(True)
                plt.savefig(os.path.join(eq_folder, f"GA_Convergence_{eq_name}.png"))
                plt.close()

    # =================================================================================
    # FINAL DASHBOARD
    # =================================================================================
    print("\n" + "#"*70)
    print(" GENERATING FINAL COMPARISON DASHBOARD")
    print("#"*70)
    
    eq_labels = [r['EQ'] for r in comparison_results]
    reductions = [r['Reduction'] for r in comparison_results]
    drift_base = [r['Drift_Base'] for r in comparison_results]
    drift_opt = [r['Drift_TMD'] for r in comparison_results]
    disp_base_m = [r['Disp_Base'] * 1000 for r in comparison_results] 
    disp_opt_m = [r['Disp_TMD'] * 1000 for r in comparison_results]   
    
    x = np.arange(len(eq_labels))
    width = 0.30

    fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(10, 15))

    def autolabel(rects, ax, fmt='{:.4f}'):
        for rect in rects:
            height = rect.get_height()
            ax.annotate(fmt.format(height),
                        xy=(rect.get_x() + rect.get_width() / 2, height),
                        xytext=(0, 3), 
                        textcoords="offset points",
                        ha='center', va='bottom', fontsize=8, rotation=90)

    # Plot 1: Drift
    rects1 = ax1.bar(x - width/2, drift_base, width, label='Without TMD', color='salmon')
    rects2 = ax1.bar(x + width/2, drift_opt, width, label='With TMD', color='skyblue')
    ax1.set_ylabel('Max Inter-Storey Drift')
    ax1.set_title('Metric 1: Maximum Drift Comparison')
    ax1.set_xticks(x); ax1.set_xticklabels(eq_labels); ax1.legend()
    ax1.grid(axis='y', linestyle='--', alpha=0.7)
    autolabel(rects1, ax1)
    autolabel(rects2, ax1)

    # Plot 2: Displacement
    rects3 = ax2.bar(x - width/2, disp_base_m, width, label='Without TMD', color='orange')
    rects4 = ax2.bar(x + width/2, disp_opt_m, width, label='With TMD', color='lightgreen')
    ax2.set_ylabel('Max Roof Disp (mm)')
    ax2.set_title('Metric 2: Maximum Roof Displacement Comparison')
    ax2.set_xticks(x); ax2.set_xticklabels(eq_labels); ax2.legend()
    ax2.grid(axis='y', linestyle='--', alpha=0.7)
    autolabel(rects3, ax2, fmt='{:.1f}')
    autolabel(rects4, ax2, fmt='{:.1f}')

    # Plot 3: Reduction
    bars = ax3.bar(x, reductions, width=0.5, color='mediumpurple')
    ax3.set_ylabel('Reduction (%)')
    ax3.set_title('Efficiency: % Reduction in Drift')
    ax3.set_xticks(x); ax3.set_xticklabels(eq_labels); ax3.grid(axis='y', linestyle='--', alpha=0.7)
    for bar in bars:
        height = bar.get_height()
        ax3.text(bar.get_x() + bar.get_width()/2., height, f'{height:.1f}%', ha='center', va='bottom')

    plt.tight_layout()
    final_plot_path = os.path.join(common_cfg["save_folder"], "FINAL_COMPARISON_ALL_METRICS.png")
    plt.savefig(final_plot_path)
    print(f"\n[DONE] Final Dashboard Saved at:\n{final_plot_path}")
