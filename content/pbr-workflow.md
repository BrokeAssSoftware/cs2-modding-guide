# PBR Asset Workflow

1. **Modeling basics**
   - Model at 1:1 scale and align pivots with placement expectations (0,0,0 at ground contact).
   - Maintain consistent texel density-target 256 px per metre for close-up assets and higher for hero pieces.
2. **Export settings**
   - Export meshes as FBX 2018 with Y-up, centimetres, and tangents.
   - Separate material slots with suffixes (`_Win`, `_Gls`, `_Gra`) to match vanilla conventions.
3. **Texture workflow**
   - Use the official Substance templates or an equivalent pipeline that outputs the CS2 channel packing:
     - `_BaseColor` - RGB albedo
     - `_MaskMap` - RGBA packed mask
     - `_ControlMask` - optional atlas control mask
     - `_Normal` - OpenGL normal map
     - `_Emissive` - emissive intensity
   - Stick to PNG or TGA textures with power-of-two resolutions (512, 1024, 2048). Atlas materials only when multiple assets share them.
4. **LOD strategy**
   - Author LOD meshes with around 60 percent fewer triangles and simplified materials.
   - Preview LOD swaps in the editor and profile them using the in-game render stats window.

