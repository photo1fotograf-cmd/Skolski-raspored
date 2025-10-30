# Skolski-raspored
raspored za ekonomsku
import pandas as pd
from ortools.sat.python import cp_model
import datetime, os
from openpyxl import Workbook
from openpyxl.styles import Alignment, PatternFill, Font, Border, Side
from openpyxl.utils import get_column_letter
from collections import defaultdict

# --------- KONFIGURACIJA ----------
INPUT = r"C:\Users\DejanPC\Desktop\Python\New folder\pregled_nastavnika_sa_sabiranjemGRUPE.xlsx"
OUT_DIR = r"C:\Users\DejanPC\Desktop\Python\New folder"
TIMESTAMP = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
OUT_PATH = os.path.join(OUT_DIR, f"raspored_auto_{TIMESTAMP}.xlsx")
DIAGNOSTIC_PATH = os.path.join(OUT_DIR, f"dijagnostika_{TIMESTAMP}.txt")

CICEVAC_ODELJENJA = {"1-8", "2-8", "3-8", "4-8"}
DANI = ["Ponedeljak", "Utorak", "Sreda", "Četvrtak", "Petak"]
CASOVI = list(range(1, 8))
SLOTS = [(d, c) for d in range(len(DANI)) for c in CASOVI]
SOLVER_TIME_SECONDS = 600
MAX_UCIONICA = 40

# ========== STRATEGIJA ==========
# 1. Prvo pokušaj sa HARD constraints
# 2. Ako ne uspe, relaksiraj neka pravila (SOFT)
# 3. Prikaži detaljnu dijagnostiku
RELAXATION_MODES = {
    "STRICT": {"group_penalty": 10000, "shift_penalty": 5000, "description": "Sve HARD"},
    "RELAX_SHIFTS": {"group_penalty": 10000, "shift_penalty": 500, "description": "Dozvoli mešanje smena"},
    "RELAX_GROUPS": {"group_penalty": 1000, "shift_penalty": 5000, "description": "Dozvoli grupe u različitim slotovima"},
    "RELAX_ALL": {"group_penalty": 100, "shift_penalty": 100, "description": "Sve SOFT"}
}

TEACHER_EXCEPTIONS = {
    "Обрадовић Обрад": {
        "forbidden_slots": [(d, 1) for d in range(5)] + [(d, 2) for d in range(5)],
        "description": "Nikad nema prva 2 časa"
    },
    "Пршић Микица": {
        "allow_shift_mixing": True,
        "description": "Može da preklapa razrede i meša smene"
    },
    "Јеврић Александра": {
        "forbidden_slots": [(0, 6), (0, 7)],
        "description": "Ponedeljak bez 6. i 7. časa"
    },
    "Миленковић Иван": {
        "allowed_days": [0],
        "description": "Radi samo ponedeljak"
    },
    "Николић Душан": {
        "forbidden_slots": [(4, c) for c in range(3, 8)],
        "description": "Petak samo prva dva časa"
    },
    "Магделинић Дамир": {
        "one_day_only": [(1, [4, 5, 6, 7]), (2, [1, 2]), (4, [1, 2, 3])],
        "description": "Samo JEDAN dan: utorak 4-7 ILI sreda 1-2 ILI petak 1-3"
    }
}

class DiagnosticLogger:
    """Logger za dijagnostiku problema"""
    def __init__(self, path):
        self.path = path
        self.logs = []
    
    def log(self, message, level="INFO"):
        timestamp = datetime.datetime.now().strftime("%H:%M:%S")
        self.logs.append(f"[{timestamp}] [{level}] {message}")
        print(f"[{level}] {message}")
    
    def save(self):
        with open(self.path, 'w', encoding='utf-8') as f:
            f.write("\n".join(self.logs))
        print(f"\n💾 Dijagnostika sačuvana: {self.path}")

def read_grouped(path, logger):
    if not os.path.exists(path):
        raise FileNotFoundError(f"Fajl ne postoji: {path}")
    
    logger.log("Učitavam Excel fajl...")
    raw = pd.read_excel(path)
    raw.columns = [str(c).strip() for c in raw.columns]
    
    raw = raw[raw["Odeljenje"].notna()].copy()
    raw = raw[raw["Nastavnik"].notna()].copy()
    raw = raw[raw["Predmet"].notna()].copy()
    
    raw["Časova nedeljno"] = pd.to_numeric(raw["Časova nedeljno"], errors='coerce').fillna(0)
    raw["Vežbi nedeljno"] = pd.to_numeric(raw["Vežbi nedeljno"], errors='coerce').fillna(0)
    raw["Ukupno časova"] = raw["Časova nedeljno"] + raw["Vežbi nedeljno"]
    
    raw = raw[raw["Ukupno časova"] > 0].copy()
    raw["Nastavnik"] = raw["Nastavnik"].astype(str).str.strip()
    raw["Odeljenje"] = raw["Odeljenje"].astype(str).str.strip()
    raw["Predmet"] = raw["Predmet"].astype(str).str.strip()
    raw["Kabinet"] = raw["Kabinet"].astype(str).str.strip()
    raw["Smena"] = raw["Odeljenje"].apply(lambda x: "Prva" if x.startswith(("1-", "3-")) else "Druga")
    
    grouped = raw.groupby(["Nastavnik", "Odeljenje", "Predmet", "Kabinet", "Vežbi nedeljno", "Smena"], as_index=False).agg({
        "Ukupno časova": "sum"
    })
    
    logger.log(f"Učitano {len(grouped)} jedinstvenih kombinacija", "SUCCESS")
    return grouped

def build_class_items(grouped):
    items = []
    for _, r in grouped.iterrows():
        total = int(r["Ukupno časova"])
        for i in range(total):
            items.append({
                "Nastavnik": str(r["Nastavnik"]).strip(),
                "Odeljenje": str(r["Odeljenje"]).strip(),
                "Predmet": str(r["Predmet"]).strip(),
                "Kabinet": str(r["Kabinet"]).strip(),
                "Vežbi nedeljno": float(r["Vežbi nedeljno"]),
                "Smena": str(r["Smena"]).strip()
            })
    return items

def get_base_odeljenje(odeljenje):
    if isinstance(odeljenje, str) and (odeljenje.endswith("g1") or odeljenje.endswith("g2")):
        return odeljenje[:-2]
    return odeljenje

def analyze_conflicts(items, logger):
    """Detaljna analiza potencijalnih konflikata"""
    logger.log("\n" + "="*60, "INFO")
    logger.log("ANALIZA KONFLIKATA", "INFO")
    logger.log("="*60, "INFO")
    
    # 1. Analiza opterećenja po nastavnicima
    teacher_hours = defaultdict(int)
    teacher_shifts = defaultdict(set)
    
    for item in items:
        teacher_hours[item["Nastavnik"]] += 1
        teacher_shifts[item["Nastavnik"]].add(item["Smena"])
    
    logger.log("\n📊 Nastavnici sa previše časova:", "WARNING")
    for teacher, hours in sorted(teacher_hours.items(), key=lambda x: -x[1])[:10]:
        available = len(SLOTS)
        if teacher in TEACHER_EXCEPTIONS:
            exc = TEACHER_EXCEPTIONS[teacher]
            if "forbidden_slots" in exc:
                available -= len(exc["forbidden_slots"])
            if "allowed_days" in exc:
                available = min(available, len(exc["allowed_days"]) * len(CASOVI))
        
        status = "⚠️ PROBLEM" if hours > available else "✓"
        logger.log(f"   {teacher:30} {hours:3} časova / {available:3} dostupno {status}", 
                  "WARNING" if hours > available else "INFO")
    
    # 2. Analiza mešanja smena
    logger.log("\n🔄 Nastavnici sa mešanjem smena:", "WARNING")
    for teacher, shifts in teacher_shifts.items():
        if len(shifts) > 1:
            allow = TEACHER_EXCEPTIONS.get(teacher, {}).get("allow_shift_mixing", False)
            status = "✓ Dozvoljeno" if allow else "⚠️ KONFLIKT"
            logger.log(f"   {teacher:30} {shifts} {status}", 
                      "INFO" if allow else "WARNING")
    
    # 3. Analiza grupa
    logger.log("\n👥 Analiza odeljenja sa grupama:", "INFO")
    bases_with_groups = set()
    for item in items:
        if item["Odeljenje"].endswith("g1") or item["Odeljenje"].endswith("g2"):
            bases_with_groups.add(get_base_odeljenje(item["Odeljenje"]))
    
    for base in sorted(bases_with_groups):
        whole_count = sum(1 for it in items if it["Odeljenje"] == base)
        g1_count = sum(1 for it in items if it["Odeljenje"] == f"{base}g1")
        g2_count = sum(1 for it in items if it["Odeljenje"] == f"{base}g2")
        
        logger.log(f"   {base}: Celo={whole_count}, g1={g1_count}, g2={g2_count}", "INFO")
        
        if g1_count != g2_count:
            logger.log(f"      ⚠️ g1 i g2 imaju različit broj časova!", "WARNING")
    
    # 4. Analiza opterećenja po slotovima
    logger.log(f"\n📈 Prosečno opterećenje: {len(items)/len(SLOTS):.2f} časova po slotu", "INFO")
    if len(items)/len(SLOTS) > MAX_UCIONICA:
        logger.log(f"   ⚠️ Prekoračenje! Max učionica: {MAX_UCIONICA}", "ERROR")

def identify_group_structure(items):
    structure = {}
    bases_with_groups = set()
    
    for item in items:
        od = item["Odeljenje"]
        if od.endswith("g1") or od.endswith("g2"):
            bases_with_groups.add(get_base_odeljenje(od))
    
    for base in bases_with_groups:
        structure[base] = {"whole": [], "g1": [], "g2": []}
    
    for idx, item in enumerate(items):
        od = item["Odeljenje"]
        base = get_base_odeljenje(od)
        
        if base in bases_with_groups:
            if od == base:
                structure[base]["whole"].append(idx)
            elif od.endswith("g1"):
                structure[base]["g1"].append(idx)
            elif od.endswith("g2"):
                structure[base]["g2"].append(idx)
    
    return structure

def solve_with_mode(items, mode_name, logger):
    """Pokušava da reši raspored sa datim relaxation modom"""
    mode = RELAXATION_MODES[mode_name]
    logger.log(f"\n🔧 Pokušavam mod: {mode_name} - {mode['description']}", "INFO")
    
    model = cp_model.CpModel()
    n = len(items)
    S = len(SLOTS)
    
    assign = {}
    for i in range(n):
        for s in range(S):
            assign[(i, s)] = model.NewBoolVar(f"a_{i}_{s}")
    
    # Svaki čas tačno jednom
    for i in range(n):
        model.Add(sum(assign[(i, s)] for s in range(S)) == 1)
    
    by_teacher = defaultdict(list)
    by_class = defaultdict(list)
    
    for i, it in enumerate(items):
        by_teacher[it["Nastavnik"]].append(i)
        by_class[it["Odeljenje"]].append(i)
    
    # Nastavnik - jedan čas u slotu
    for teacher, idxs in by_teacher.items():
        for s in range(S):
            model.Add(sum(assign[(i, s)] for i in idxs) <= 1)
    
    # Odeljenje - jedan čas u slotu
    for cls, idxs in by_class.items():
        for s in range(S):
            model.Add(sum(assign[(i, s)] for i in idxs) <= 1)
    
    # Broj učionica
    for s in range(S):
        model.Add(sum(assign[(i, s)] for i in range(n)) <= MAX_UCIONICA)
    
    # Razdvajanje škola (HARD)
    for teacher, idxs in by_teacher.items():
        for d in range(len(DANI)):
            cic_slots = []
            krus_slots = []
            
            for i in idxs:
                base_od = get_base_odeljenje(items[i]["Odeljenje"])
                for s in range(S):
                    if SLOTS[s][0] == d:
                        if base_od in CICEVAC_ODELJENJA:
                            cic_slots.append(assign[(i, s)])
                        else:
                            krus_slots.append(assign[(i, s)])
            
            if cic_slots and krus_slots:
                has_cic = model.NewBoolVar(f"cic_{teacher}_{d}")
                has_krus = model.NewBoolVar(f"krus_{teacher}_{d}")
                
                model.Add(sum(cic_slots) >= 1).OnlyEnforceIf(has_cic)
                model.Add(sum(cic_slots) == 0).OnlyEnforceIf(has_cic.Not())
                model.Add(sum(krus_slots) >= 1).OnlyEnforceIf(has_krus)
                model.Add(sum(krus_slots) == 0).OnlyEnforceIf(has_krus.Not())
                
                model.Add(has_cic + has_krus <= 1)
    
    penalties = []
    group_structure = identify_group_structure(items)
    
    # Pravila za grupe
    for base, structure in group_structure.items():
        whole_idxs = structure["whole"]
        g1_idxs = structure["g1"]
        g2_idxs = structure["g2"]
        
        # g1 i g2 moraju biti zajedno
        if g1_idxs and g2_idxs:
            for i1 in g1_idxs:
                for i2 in g2_idxs:
                    for s in range(S):
                        different = model.NewBoolVar(f"diff_{i1}_{i2}_{s}")
                        model.Add(assign[(i1, s)] != assign[(i2, s)]).OnlyEnforceIf(different)
                        model.Add(assign[(i1, s)] == assign[(i2, s)]).OnlyEnforceIf(different.Not())
                        penalties.append(different * mode["group_penalty"])
        
        # Celo odeljenje ne sme biti sa grupama
        if whole_idxs and (g1_idxs or g2_idxs):
            group_idxs = g1_idxs + g2_idxs
            for s in range(S):
                whole_in_slot = [assign[(i, s)] for i in whole_idxs]
                groups_in_slot = [assign[(i, s)] for i in group_idxs]
                
                if whole_in_slot and groups_in_slot:
                    has_whole = model.NewBoolVar(f"whole_{base}_{s}")
                    has_groups = model.NewBoolVar(f"groups_{base}_{s}")
                    
                    model.Add(sum(whole_in_slot) >= 1).OnlyEnforceIf(has_whole)
                    model.Add(sum(whole_in_slot) == 0).OnlyEnforceIf(has_whole.Not())
                    model.Add(sum(groups_in_slot) >= 1).OnlyEnforceIf(has_groups)
                    model.Add(sum(groups_in_slot) == 0).OnlyEnforceIf(has_groups.Not())
                    
                    both = model.NewBoolVar(f"both_{base}_{s}")
                    model.Add(both <= has_whole)
                    model.Add(both <= has_groups)
                    model.Add(both >= has_whole + has_groups - 1)
                    
                    penalties.append(both * mode["group_penalty"])
    
    # Mešanje smena
    for teacher, idxs in by_teacher.items():
        if TEACHER_EXCEPTIONS.get(teacher, {}).get("allow_shift_mixing"):
            continue
        
        for d in range(len(DANI)):
            shift_1 = []
            shift_2 = []
            
            for i in idxs:
                for s in range(S):
                    if SLOTS[s][0] == d:
                        if items[i]["Smena"] == "Prva":
                            shift_1.append(assign[(i, s)])
                        else:
                            shift_2.append(assign[(i, s)])
            
            if shift_1 and shift_2:
                has_1 = model.NewBoolVar(f"s1_{teacher}_{d}")
                has_2 = model.NewBoolVar(f"s2_{teacher}_{d}")
                
                model.Add(sum(shift_1) >= 1).OnlyEnforceIf(has_1)
                model.Add(sum(shift_1) == 0).OnlyEnforceIf(has_1.Not())
                model.Add(sum(shift_2) >= 1).OnlyEnforceIf(has_2)
                model.Add(sum(shift_2) == 0).OnlyEnforceIf(has_2.Not())
                
                both = model.NewBoolVar(f"mix_{teacher}_{d}")
                model.Add(both <= has_1)
                model.Add(both <= has_2)
                model.Add(both >= has_1 + has_2 - 1)
                
                penalties.append(both * mode["shift_penalty"])
    
    # Primena izuzetaka
    for teacher, idxs in by_teacher.items():
        if teacher not in TEACHER_EXCEPTIONS:
            continue
        
        exc = TEACHER_EXCEPTIONS[teacher]
        
        if "forbidden_slots" in exc:
            for day_idx, cas_num in exc["forbidden_slots"]:
                for s in range(S):
                    if SLOTS[s] == (day_idx, cas_num):
                        for i in idxs:
                            model.Add(assign[(i, s)] == 0)
        
        if "allowed_days" in exc:
            for i in idxs:
                for s in range(S):
                    if SLOTS[s][0] not in exc["allowed_days"]:
                        model.Add(assign[(i, s)] == 0)
    
    # Optimizacija
    cost = []
    for i in range(n):
        for s in range(S):
            d, c = SLOTS[s]
            weight = d * 5 + c
            cost.append(weight * assign[(i, s)])
    
    model.Minimize(sum(penalties) + sum(cost))
    
    solver = cp_model.CpSolver()
    solver.parameters.max_time_in_seconds = SOLVER_TIME_SECONDS // len(RELAXATION_MODES)
    solver.parameters.num_search_workers = 8
    
    status = solver.Solve(model)
    
    if status in (cp_model.OPTIMAL, cp_model.FEASIBLE):
        logger.log(f"✅ Rešenje pronađeno u modu: {mode_name}", "SUCCESS")
        
        rows = []
        for i in range(n):
            for s in range(S):
                if solver.Value(assign[(i, s)]) == 1:
                    d_idx, cas_num = SLOTS[s]
                    rows.append({
                        "Nastavnik": items[i]["Nastavnik"],
                        "Odeljenje": items[i]["Odeljenje"],
                        "Predmet": items[i]["Predmet"],
                        "Dan": DANI[d_idx],
                        "Čas": cas_num,
                        "Vežbi nedeljno": items[i]["Vežbi nedeljno"],
                        "Kabinet": items[i]["Kabinet"],
                        "Smena": items[i]["Smena"]
                    })
                    break
        
        return pd.DataFrame(rows), mode_name
    
    logger.log(f"❌ Nema rešenja u modu: {mode_name}", "WARNING")
    return None, None

def export_stylized(df, out_path, logger):
    """Eksportuje raspored u formatiran Excel"""
    logger.log(f"Kreiram Excel sa {len(df)} časova...", "INFO")
    
    wb = Workbook()
    ws = wb.active
    ws.title = "Распоред часова"
    
    # Stilovi
    center = Alignment(horizontal="center", vertical="center", wrap_text=True)
    left = Alignment(horizontal="left", vertical="center")
    bold_font = Font(bold=True, size=11)
    header_fill = PatternFill(start_color="4472C4", end_color="4472C4", fill_type="solid")
    header_font = Font(bold=True, color="FFFFFF", size=11)
    
    gray = PatternFill(start_color="E8E8E8", end_color="E8E8E8", fill_type="solid")
    red = PatternFill(start_color="FFE5E5", end_color="FFE5E5", fill_type="solid")
    yellow = PatternFill(start_color="FFF4CC", end_color="FFF4CC", fill_type="solid")
    blue = PatternFill(start_color="ADD8E6", end_color="ADD8E6", fill_type="solid")
    
    thin_border = Border(
        left=Side(style='thin'),
        right=Side(style='thin'),
        top=Side(style='thin'),
        bottom=Side(style='thin')
    )
    
    # Header
    ws.cell(1, 1, "R.br").font = header_font
    ws.cell(1, 1).fill = header_fill
    ws.cell(1, 1).alignment = center
    
    ws.cell(1, 2, "Nastavnik").font = header_font
    ws.cell(1, 2).fill = header_fill
    ws.cell(1, 2).alignment = center
    
    ws.cell(1, 3, "Ukupno").font = header_font
    ws.cell(1, 3).fill = header_fill
    ws.cell(1, 3).alignment = center
    
    # Dani i časovi
    col = 4
    empty_columns = []
    day_start_columns = {}
    
    for idx, dan in enumerate(DANI):
        if idx > 0 or idx == 0:
            empty_columns.append(col)
            col += 1
        
        day_start_columns[dan] = col
        
        ws.merge_cells(start_row=1, start_column=col, end_row=1, end_column=col + 6)
        cell = ws.cell(1, col, dan.upper())
        cell.font = header_font
        cell.fill = header_fill
        cell.alignment = center
        
        for i, cas in enumerate(CASOVI):
            cell = ws.cell(2, col + i, str(cas))
            cell.alignment = center
            cell.font = bold_font
            cell.fill = gray
            cell.border = thin_border
        
        col += 7
    
    # Sivljenje praznjih kolona
    for col_idx in empty_columns:
        for row_idx in range(1, 200):
            cell = ws.cell(row_idx, col_idx)
            cell.fill = gray
            cell.border = thin_border
    
    # Popunjavanje podataka
    teachers = sorted(df["Nastavnik"].unique())
    teacher_counts = df.groupby("Nastavnik").size().to_dict()
    
    row = 3
    for idx, teacher in enumerate(teachers, 1):
        ws.cell(row, 1, idx).alignment = center
        ws.cell(row, 2, teacher).alignment = left
        ws.cell(row, 3, teacher_counts.get(teacher, 0)).alignment = center
        
        tdf = df[df["Nastavnik"] == teacher]
        mapping = {}
        
        for _, r in tdf.iterrows():
            key = (r["Dan"], int(r["Čas"]))
            if key not in mapping:
                mapping[key] = []
            mapping[key].append((r["Odeljenje"], r["Vežbi nedeljno"], r["Kabinet"]))
        
        for dan in DANI:
            col_start = day_start_columns[dan]
            
            for cas in CASOVI:
                odeljenja_data = mapping.get((dan, cas), [])
                odeljenja = [od for od, _, _ in odeljenja_data]
                text = ", ".join(odeljenja) if odeljenja else ""
                
                cell = ws.cell(row, col_start + cas - 1, text)
                cell.alignment = center
                cell.border = thin_border
                
                if odeljenja:
                    # Provera da li su sva odeljenja iz Ćićevca
                    all_cic = all(get_base_odeljenje(od) in CICEVAC_ODELJENJA for od in odeljenja)
                    any_cic = any(get_base_odeljenje(od) in CICEVAC_ODELJENJA for od in odeljenja)
                    
                    # Provera da li su vežbe
                    has_vezbe = any(vezbe > 0 and kab.lower() == "кабинет" 
                                   for _, vezbe, kab in odeljenja_data)
                    
                    # Bojenje
                    if all_cic:
                        cell.fill = red  # Ćićevac - crveno
                    elif any_cic:
                        cell.fill = yellow  # Mešovito - žuto
                    elif has_vezbe:
                        cell.fill = blue  # Vežbe - plavo
                    else:
                        cell.fill = gray  # Normalno - sivo
        
        row += 1
    
    # Podešavanje širina kolona
    ws.column_dimensions['A'].width = 6
    ws.column_dimensions['B'].width = 35
    ws.column_dimensions['C'].width = 8
    
    for col_idx in range(4, col + 1):
        ws.column_dimensions[get_column_letter(col_idx)].width = 12
    
    wb.save(out_path)
    logger.log(f"✅ Excel sačuvan: {out_path}", "SUCCESS")

def main():
    print("=" * 70)
    print("HIBRIDNI GENERATOR RASPOREDA")
    print("=" * 70)
    
    logger = DiagnosticLogger(DIAGNOSTIC_PATH)
    
    try:
        grouped = read_grouped(INPUT, logger)
        items = build_class_items(grouped)
        
        logger.log(f"\n📋 Ukupno časova za raspored: {len(items)}", "INFO")
        logger.log(f"📋 Ukupno slotova: {len(SLOTS)}", "INFO")
        logger.log(f"📋 Prosečno po slotu: {len(items) / len(SLOTS):.2f}", "INFO")
        
        analyze_conflicts(items, logger)
        
        # Pokušaj sa različitim modovima
        logger.log("\n" + "="*60, "INFO")
        logger.log("POKUŠAVAM GENERISANJE SA RAZLIČITIM STRATEGIJAMA", "INFO")
        logger.log("="*60, "INFO")
        
        df = None
        used_mode = None
        
        for mode_name in ["STRICT", "RELAX_SHIFTS", "RELAX_GROUPS", "RELAX_ALL"]:
            df, used_mode = solve_with_mode(items, mode_name, logger)
            if df is not None:
                break
        
        if df is None:
            logger.log("\n❌ NIJE PRONAĐENO REŠENJE NI SA JEDNOM STRATEGIJOM!", "ERROR")
            logger.save()
            return
        
        logger.log(f"\n✅ Korišćena strategija: {used_mode}", "SUCCESS")
        logger.log(f"✅ Generisano {len(df)} časova", "SUCCESS")
        
        # Eksportuj (koristi postojeću funkciju)
        logger.log(f"\n💾 Eksportujem u Excel...", "INFO")
        export_stylized(df, OUT_PATH, logger)
        
        logger.save()
        
        print("\n" + "=" * 70)
        print("✅ USPEŠNO ZAVRŠENO!")
        print(f"📄 Raspored: {OUT_PATH}")
        print(f"📊 Dijagnostika: {DIAGNOSTIC_PATH}")
        print("=" * 70)
        
    except Exception as e:
        logger.log(f"\n❌ KRITIČNA GREŠKA: {str(e)}", "ERROR")
        logger.save()
        raise

if __name__ == "__main__":
    main()
