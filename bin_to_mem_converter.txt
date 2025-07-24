# convert_ascii_bin_to_mem.py

# Input file containing lines like "00010111"
input_file = "Vexii_trial1/MicroSoc.v_toplevel_system_ram_thread_logic_mem_symbol3.bin"

# Output file with actual hex values (for use in Verilog $readmemh)
output_file = "Vexii_trial1/program3.mem"

# Optional: Output binary file if needed
binary_output_file = "output.bin"

with open(input_file, "r") as fin:
    lines = fin.readlines()

# Clean up and process each binary string
bytes_list = []
for line in lines:
    bin_str = line.strip()
    if bin_str:  # Skip empty lines
        if len(bin_str) != 8 or any(c not in '01' for c in bin_str):
            raise ValueError(f"Invalid binary string: {bin_str}")
        value = int(bin_str, 2)       # Convert to integer
        bytes_list.append(value)

# Write hex format for Verilog memory initialization (.mem)
with open(output_file, "w") as fout:
    for b in bytes_list:
        fout.write(f"{b:02X}\n")      # Write as two-digit hex

# Optional: Write raw binary file (e.g., for SPI flash)
with open(binary_output_file, "wb") as fbin:
    fbin.write(bytearray(bytes_list))

print(f"Converted {len(bytes_list)} binary lines to .mem and .bin formats.")
