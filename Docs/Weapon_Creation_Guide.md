# Creación de armas en Arma Reforger — Guía de proceso

Resumen de las páginas oficiales de la wiki de Bohemia (copias locales en
`D:\Bibliotecas\Documentos\ArgA\Reforger\Documentacion\Wiki`):

- Weapon Creation (índice)
- Weapon Creation / Asset Preparation
- Weapon Creation / Prefab Configuration
- Weapon Animation (índice)
- Weapon Animation / Setup
- Weapon Animation / Basic Tutorial
- Weapon Animation / Advanced Tutorial — **NO descargada todavía; no resumida aquí**

Archivos de ejemplo oficiales: repositorio *Arma Reforger Samples* en GitHub
(SampleWeapon_01, `Character_AnimationRig_Example.blend`, `BlenderPoses.blend`).

---

## 1. Preguntas clave

**¿Hace falta un modelo 3D?** Sí. El flujo es FBX → Workbench (`Register and Import`)
→ `.xob` + materiales `.emat`. Se puede modelar desde cero, reutilizar un modelo o
importar un P3D de Arma 3 con las Enfusion Blender Tools. No es obligatorio usar
Blender (3ds Max y Maya también sirven), pero la guía está escrita para Blender.

**¿Se puede dividir en partes?** Sí, y es lo recomendado: *"Enfusion allows you to
assemble weapon from multiple parts"*. Cada parte va en su propio FBX. El arma lleva
puntos `slot_*` y cada accesorio lleva su contraparte `snap_*`. Ejemplo oficial:
cuerpo, empuñadura, miras metálicas y cargador como FBX separados.

**Texturas:** las de Arma 3 son spec-gloss y no se convierten bien a PBR
metal/rough. Lo normal es rehacerlas (Substance Painter). Material estándar:
`_BCR` (base color + roughness) y `_NMO` (normal map).

---

## 2. Fase 1 — Asset Preparation (Blender)

1. **Orientación:** todo apuntando a +Y en Blender y 3ds Max (+Z en Maya).
2. **Estructura de carpetas:** seguir la convención oficial *Data (file) structure*
   para que los plugins de automatización lean bien los assets.
3. **Nombres con significado para el motor:**
   - `_LODx` — niveles de detalle.
   - `UBX_`, `UCX_`, `USP_`, `UCS_`, `UCL_`, `UTM_` — colliders.
   - `OCC_` — oclusores de geometría.
4. **Slots y snap points** como Empties (Plain Axis). La rotación del empty define
   el eje; ya no hacen falta dos puntos como en Arma 3. Tip: crear los slots con la
   malla aún entera, copiar malla + empties a una escena nueva y renombrar
   `slot_XX` a `snap_XX`.
   - Slots del ejemplo: `slot_ironsight_front`, `slot_ironsight_rear`,
     `slot_magazine`, `slot_optics`, `slot_underbarrel`.
   - Otros empties: `snap_hand_right`, `snap_hand_left` (IK de manos en deployment),
     `barrel_chamber`, `barrel_muzzle` (origen y dirección de la bala en
     `MuzzleComponent`), `eye` (vista de mira en `SightsComponent`).
5. **Colliders:**
   - Mínimo uno con layer preset `Weapon` (sin él faltan las acciones de equipar).
   - `FireGeo` aparte si se quiere más detalle (puede ser trimesh); el de `Weapon`
     debe ser simple (convexo). Si la geometría es simple, uno solo sirve para ambos.
   - Cada collider necesita la propiedad `usage` y el material correcto.
   - Herramienta: *Objects Tool* de las Enfusion Blender Tools para asignar
     materiales de juego y layer presets.
   - Verificación: *Model Quality Assurance* → opción UCX Collider.
6. **Skeleton (Armature):**
   - Llamarlo exactamente `Armature` (evita un hueso artificial al importar).
   - En 0,0,0 y escala 1.0.
   - Hueso raíz: `w_root`, orientado hacia el frente del arma.
   - Huesos de partes móviles hijos de `w_root`: `w_fire_mode`, `w_trigger`,
     `w_charging_handle`, `w_bolt`. Pivotes en el centro real del movimiento.
   - Mantener orientación Y+. Usar nombres vanilla para poder usar los perfiles de
     exportación de animación vanilla.
7. **Skinning:**
   - Armature modifier + vertex groups con el nombre del hueso.
   - **Toda** la malla debe estar pesada a algún hueso; lo que no se mueve va a
     `w_root`. Si no, el importador avisa: `Missing some mesh skinning weights ...
     Weighting them to root`.
   - Colliders animados: solo los trimesh pueden tener skinning; el resto, 100% de
     peso (en Blender requiere Object Relations).
   - Conviene hacer hijos del Armature a los slots (facilita encajar el cargador).
8. **Export FBX (Blender):**
   - Object Types: Empty + Armature + Mesh.
   - **Custom Properties activado** (si no, se pierden los LayerPresets).
   - **Leaf Bones desactivado.**
   - 3ds Max: FBX binario 2014/2015; Triangulation y Preserve Edge Orientation off.
9. **Import en Workbench:**
   - Clic derecho en el FBX → `Register and Import`.
   - Revisar colliders y materiales (si no encuentra la textura, crea un `.emat`
     "dummy" junto al FBX).
   - Piezas con huesos: activar `Export Skinning` (en `Import Settings`, sección
     Miscellaneous). Piezas con solo empties (cargador): `Export Scene Hierarchy`.
   - Los cambios en Import Settings solo se aplican tras `Reimport resource (PC)`.
   - Materiales: arrastrar el `.emat` sobre el icono de material, o cambiar
     `Material Assign` en Import Settings y reimportar.

---

## 3. Fase 2 — Prefab Configuration (Workbench)

1. **Prefab base:** clic derecho en `Prefabs/Weapons/Core/Rifle_Base.et` →
   `Inherit in`, nombre `NombreArma_base`, moverlo (por ejemplo a
   `Prefabs/Weapons/Rifles`). Alternativa: duplicar un arma parecida
   (`Rifle_M16A2_base.et`, etc.).
2. **MeshObject:** asignar el `.xob`.
3. **Componentes que faltan al heredar de `Rifle_Base`:**
   - `RigidBody` con *Model Geometry* activado (sin él no se puede recoger el arma).
   - `SCR_MeleeWeaponProperties` (daño cuerpo a cuerpo; ~45 para un rifle).
4. **Efectos en `MuzzleComponent`:**
   - `CaseEjectingEffectComponent` hijo para humo (por ejemplo `Smoke_M16A2.ptc`).
     Effect Position: clase `EntitySlotInfo`, Pivot ID `barrel_muzzle`, Has Sound
     desactivado.
   - Casquillos: partícula (por ejemplo `Casing_556x45.ptc`).
   - Fogonazo: `SCR_MuzzleEffectComponent` (por ejemplo `Muzzle_AK74.ptc`).
5. **Inventario** (`SCR_WeaponAttachmentsStorageComponent`):
   - Item Display Name (nombre y descripción; localizable).
   - Item Phys Attributes (usar *set class*): peso en kg, *Size Setup Strategy* =
     Volume, *Item Volume* en cm3 (AK74 ≈ 1500, ametralladoras ≈ 3000).
   - Custom Attributes: añadir `ItemOneHandAnimAttributes` (valores por defecto
     suelen servir).
   - `PreviewRenderAttributes`: el inventario renderiza el modelo 3D; ajustar
     *Camera Distance To Item*. Refrescar con Shift+R en World Editor.
   - **Aviso:** la animation graph vanilla deforma el arma salvo que las posiciones
     de huesos sean EXACTAMENTE las del arma vanilla.
6. **Action Contexts:** zonas de interacción (añadir, mover, ajustar radio; se
   depura con `ActionsManagerComponent`).
7. **Modos de fuego** (`MuzzleComponent` → Fire Modes): `Rifle_Base` trae Safe y
   Single. `BaseFireMode` con `Max Burst = 1` = single; `-1` = automático; `>1` =
   ráfaga, con Burst Type: *Uninterruptible*, *Interruptible* (con memoria),
   *InterruptibleAndReseting* (sin memoria).
8. **Retroceso:** `Rifle_Base` no trae `RecoilWeaponAimModifier` ni
   `ZeroingWeaponAimModifier`; hay que añadirlos en Weapon Aim Modifiers. Zeroing
   no necesita ajustes; el retroceso se configura a mano (Linear, Angular, Turn
   Offset). Referencia: prefab del AK74.
9. **Obstrucción** (arma contra paredes, 3 fases + modificadores por accesorio) y
   **Deployment** (apoyo base y bípode, puntos de despliegue, IK targets, aim
   modifiers).
10. **Zeroing y dispersión.**
11. **Munición:** configuración del ammo, balística (arrastre, coeficiente
    balístico G1/G7, tabla balística, velocidad según longitud de cañón),
    penetración (game materials), trazadoras, daño y selección de objetivo de la IA
    (tablas balísticas de IA).
12. **Magazine well:** si el cargador es único, nueva clase de script que hereda de
    `BaseMagazineWell` (por ejemplo en `Scripts/Game/Weapon`, prefijo
    `TAG_MagazineWell_`). Recompilar con Shift+F7 (Script Editor) o Ctrl+R
    (World Editor). Asignarla en `MagazineComponent` del cargador y en
    `MuzzleComponent` del arma.
13. **Prefab de cargador:** heredar de `Magazine_Base.et`; asignar mesh,
    `MagazineComponent` (munición, capacidad) e `InventoryItemComponent`. Variante
    con trazadoras.
14. **Cajas:** arsenal (por Entity Catalogs en configs de facción, por ejemplo
    `USSR_InventoryItems.conf`) o cajas normales con cantidad exacta.

---

## 4. Fase 3 — Weapon Animation

Prerrequisitos: arma con skinning y prefab configurados; Blender 3.6 LTS o superior
con la última versión de las Enfusion Blender Tools.

### 4.1 Setup (Animation Editor, Workbench)

1. **Workspace:** abrir un workspace de arma parecida (`ak74.aw`) y usar
   `File → Duplicate Workspace`. Ubicación sugerida:
   `Assets/Weapons/Rifles/Workspaces/`. Alternativa: `File → New Workspace` con
   *Create empty* marcado.
2. **Limpiar lo copiado:** seleccionar entradas del `.asi` del arma → clic derecho →
   `Unassign animations`. Si no, las animaciones del AK74 mueven mal las piezas.
3. **Archivos (nombre del arma como base):**

   | Extensión | Contenido |
   |---|---|
   | `.aw` | workspace |
   | `.ast` | template de animación (grupos, columnas, filas) |
   | `.agr` | grafo: variables, comandos, IK chains |
   | `.agf` | archivo de grafo: hojas (sheets) con nodos; hoja principal "Master" |
   | `_weapon.asi` | instancia del arma |
   | `_player.asi` | instancia del personaje |

4. **Template:** nuevo, existente (sin poder modificarlo; depende de contenido
   vanilla) o duplicado (recomendado si se parte del AK74).
5. **Grafo:** igual; duplicar suele ser lo más rápido. Tras duplicar el `.agr`,
   quitar la referencia al `.agf` viejo y reemplazar a mano la línea
   `AnimSetTemplate` del `.agr` por la del `.aw` (editor de texto) y reiniciar
   Workbench. Los nodos se pueden copiar y pegar entre dos instancias del editor.
6. **Dos instancias:** arma y personaje se reproducen a la vez, así que sus
   animaciones deben durar lo mismo (hay excepciones).
7. **Preview models:** arma (+ cargador como modelo hijo, enganchado a
   `slot_magazine`); personaje (`AnimTestChar_USSR_01.xob` o
   `AnimTestChar_US_01.xob`) con el arma en *Right Hand Prop* y el cargador en
   *Left Hand Prop*. Se activan con doble clic o `Set as current`.
8. **Enlazar al prefab:** añadir `WeaponAnimationComponent` como hijo de
   `WeaponComponent` y configurar:
   - Anim Graph: el `.agr`.
   - Anim Instance: el `_weapon.asi`.
   - **Always Active** y **Bind With Injection** activados.
   - Anim Injection (*set class*): mismo `.agr`, Animation Instance = `_player.asi`,
     Binding Name = `Weapon`.

### 4.2 Basic Tutorial (Blender)

**Preparación**
- Workbench: `Options → Workbench → Enable net API` (sin esto Blender no puede
  exportar TXA).
- Blender: Enfusion Blender Tools con *Export Profile Folder* configurado
  (obligatorio); activar el addon *Animation: Pose Library*.
- Usar `Character_AnimationRig_Example.blend`. El esqueleto del personaje de
  Reforger NO se arregla con "Force Connect Children" ni "Automatic Bone
  Orientation" y no se modifica. Se anima únicamente el **rig Rigify**; el
  `Character` armature queda intacto y recibe el movimiento por constraints.
- Colores de Rigify: rojo = IK, verde = FK, azul = ajustes, amarillo = torso y
  cara, morado = root motion, naranja = dedos. Slider IK-FK: 0 = IK, 1 = FK.
- Poses de referencia (`BlenderPoses.blend`, añadido como Asset Library en
  Preferences → File Paths): Rifle, Pistol, LMG, RPG y LAW.

**Escena**
- Arma: escala 1 en armature y empties; slots hijos del armature; copiar al blend
  del personaje; renombrar el armature a `Weapon`.
- Anclaje (método 1 usado en el tutorial): mallas con padre `Weapon`; `Weapon`
  con constraint `Copy Transforms` hacia `Character` / hueso `RightHandProp`.
  Método 2: constraint en el hueso `w_root` más un `Weapon_Helper`.
- Cargador: anclar a `LeftHandProp`; en el rig, `prop.L` con `Copy Transforms`
  hacia `slot_magazine` (importante para deployment). Nombrar los constraints
  (por ejemplo "Snap Weapon") desde el principio.

**IK de agarre (primera animación)**
- Acción `p_<arma>_ik`, frame 0, pose base `Pose Rifle Rigify`, ajustar manos
  (`hand_ik.L/R`) y dedos.
- Exportar **una acción a tres TXA** (Enfusion Tools → Export → Enfusion Animation):

  | Archivo | Export profile |
  |---|---|
  | `p_rfl_<arma>_ik` | `501_IK` |
  | `p_rfl_<arma>_offset` | `101_FullBodyABS` |
  | `p_rfl_<arma>_safety` | `502_IK_RightHand` |

  Los perfiles exportan solo ciertos huesos; sin ellos las animaciones se pisarían.
  Armature name = Character; Secondary armature = Rig. Guardar en
  `Assets/Weapons/Rifles/<Arma>/Anims`.
- Registrar e importar los TXA (se convierten a `.anm`); previsualizar en Resource
  Manager con `AnimTestChar_*`.
- En `_player.asi`: `safety` en Idle y Safety (columnas Erc y Pne), `offset` en
  IKOffset. Provisionalmente `offset` también en Switch_Mode_On/Off (reemplazar
  después para evitar glitches al cambiar de modo).
- En el prefab: `SCR_WeaponAttachmentsStorageComponent → Item Animation
  Attributes → Animation IK Pose` = `_ik.anm`. Si el tipo de arma es otro, cambiar
  también *Animation Instance* (machinegun, RPG, LAW, pistol) con su pose base.
- Probar en juego (por ejemplo escenario `Assets_Showcase_Basic`).

**Mecánicas del arma** (en el blend del arma o en el combinado)

| Acción | Contenido | Export |
|---|---|---|
| `w_<arma>_fire` | gatillo (~25°) y cerrojo (3 frames: cerrado, abierto, cerrado) | `_trigger` con `721_Weapon_Trigger`; `_bolt_pose` con `724_Weapon_Bolt`; `_fire` con `726_Weapon_Bolt_Trigger` |
| `w_<arma>_safety` | un frame por modo de fuego (safe, semi, auto) | `_safety` con `701_Rifle_Idle` |

- Asignar en `_weapon.asi` (Erc y Pne): Trigger, BoltPose, Fire, Safety.
- Ajuste del grafo: el AK74 va Safe → Auto → Semi. Si el selector es Safe → Semi →
  Auto, cambiar el `Time` del nodo `SemiPose` (1 → 0.5) y de `AutoPose` (0.5 → 1).
- Probar el grafo en Animation Editor (variable `State`: -1 safe, 0 single, 1
  ráfaga, 2 auto).

**Limitación declarada por el propio tutorial:** con esto NO funcionan todavía
recarga, inspección ni cambio de modo completo.

### 4.3 Advanced Tutorial (pendiente)

Según el índice: animación aditiva, variantes en prone, retargeting con datos
fuente de Reforger y más. **La página no está en la carpeta de la wiki; falta
descargarla.**

Vídeos de apoyo: Modding Bootcamp #8, #9 y #10 (Animation Workshop 1–3) y
"Arma Reforger NLA Editor" (comunidad).

---

## 5. Orden de trabajo resumido

1. Modelo en partes + huesos + slots + colliders → FBX.
2. Import, colliders y materiales en Workbench.
3. Prefab heredado de `Rifle_Base.et` y su configuración (inventario, fuego,
   retroceso, etc.).
4. Munición, magazine well y prefab de cargador.
5. Workspace de animación (duplicado de arma similar) y enlace al prefab.
6. IK de agarre y mecánicas básicas en Blender.
7. Animaciones restantes (recarga, inspección, cambio de modo) — ver Advanced.
8. Cajas / arsenal para obtener el arma en juego.
