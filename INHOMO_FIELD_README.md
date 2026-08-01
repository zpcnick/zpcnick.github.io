# Octopus 8.4 非均匀电场功能修改说明

本说明文档记录在 Octopus 8.4 的 TD(含时密度泛函)任务中加入**空间非均匀外加电场**的全部修改内容,涵盖三种物理模型:泰勒展开、指数衰减、球形金属纳米粒子 Mie 偶极近场。

---

## 一、修改文件清单(共 6 个)

### A. 核心计算文件(3 个,实现势能计算)

| 文件路径 | 修改内容 |
|---|---|
| `src/hamiltonian/lasers.F90` | 新增 `E_FIELD_INHOMO_ELECTRIC` 类型与 `INHOMO_TAYLOR/EXPONENTIAL/DIPOLE` 三种模型常量;扩展 `laser_t` 结构体加入所有非均匀场参数;在 `laser_init` 中解析新输入参数;在 `laser_potential` 中实现三种模型的标量势计算;在 `laser_write_info` 中输出模型信息。 |
| `src/hamiltonian/lasers_inc.F90` | 在 `vlaser_operator_linear` 的 select case 中将 `E_FIELD_INHOMO_ELECTRIC` 与 `E_FIELD_ELECTRIC` 合并到同一分支,使其走标量势处理路径(调用 `laser_potential` 后加负号)。 |
| `src/hamiltonian/hamiltonian.F90` | 在 `epot_generate` 和 `hamiltonian_update` 的激光循环 select case 中加入 `E_FIELD_INHOMO_ELECTRIC`,确保新电场类型调用 `laser_potential`。 |

### B. 势能输出文件(3 个,实现空间分布可视化)

| 文件路径 | 修改内容 |
|---|---|
| `src/system/output.F90` | 在 `Output` 选项中新增 `external_td_potential` (bit 26),用于控制激光势能空间分布的输出。 |
| `src/system/output_h_inc.F90` | 实现 `output_scalar_pot` 子例程,按 `OutputInterval` 调用并写出激光势能到文件(支持平面切片 / Cube 格式)。 |
| `src/td/td_write.F90` | 在 TD 输出流程中按迭代步调用 `output_scalar_pot`,实现随时间演化的势能输出。 |

> **部署提示**:若仅需非均匀电场计算功能,上传前 3 个核心文件即可;若需要可视化验证势能空间分布,6 个文件全部上传。

---

## 二、三种非均匀电场模型

### 模型 1:泰勒展开(Taylor expansion)

**电场形式**(沿极化方向展开,仅对角元):
$$E_i(\mathbf{r},t) = \mathrm{pol}_i \cdot \left[ 1 + G_i \cdot \Delta r_i + \tfrac{1}{2} H_i \cdot \Delta r_i^2 \right] \cdot f(t)$$

**对应标量势**(对电场积分,代码中实际实现):
$$V(\mathbf{r},t) = f(t) \sum_{i} \mathrm{pol}_i \left[ \Delta r_i + \tfrac{1}{2} G_i \Delta r_i^2 + \tfrac{1}{6} H_i \Delta r_i^3 \right]$$

其中 $\Delta r_i = r_i - r_{0,i}$ 为相对展开中心 `r0` 的位移。

**退化条件**: $$G_i = H_i = 0$$ 时退化为均匀电场。

**输入参数**(共 9 个,均使用原子单位 a₀):
- `r0_x, r0_y, r0_z`:展开中心坐标
- `G_x, G_y, G_z`:一阶梯度(长度倒数 a₀⁻¹)
- `H_x, H_y, H_z`:二阶梯度(长度平方倒数 a₀⁻²)

---

### 模型 2:指数衰减(Exponential decay)

**电场形式**:
$$E_i(\mathbf{r},t) = \mathrm{pol}_i \cdot \exp(\alpha_i \cdot \Delta r_i) \cdot f(t)$$

**对应标量势**(解析积分, $\alpha_i \neq 0$):
$$V(\mathbf{r},t) = f(t) \sum_{i} \frac{\mathrm{pol}_i}{\alpha_i} \left[ \exp(\alpha_i \Delta r_i) - 1 \right]$$

**退化条件**: $\alpha_i \to 0$ 时, $(\exp(\alpha_i \Delta r_i) - 1)/\alpha_i \to \Delta r_i$,退化为均匀电场。代码中显式判断 `|alpha| < epsilon` 走退化分支。

**输入参数**(共 6 个,原子单位):
- `r0_x, r0_y, r0_z`:衰减中心
- `alpha_x, alpha_y, alpha_z`:衰减因子(负值表示衰减,a₀⁻¹)

---

### 模型 3:球形金属纳米粒子 Mie 偶极近场

基于 Mie 理论 n=1 偶极近似,准静态极限,严格满足 Maxwell 边界条件。

**物理量定义**:
- 介电函数 $\varepsilon = \varepsilon_\mathrm{re} + i \varepsilon_\mathrm{im}$(复数, $\varepsilon_\mathrm{im} \geq 0$)
- 散射增强因子: $\beta = \dfrac{\varepsilon - 1}{\varepsilon + 2}$ (复数,含相位延迟)
- 球内场系数: $k_\mathrm{in} = \dfrac{3}{\varepsilon + 2}$ (Mie 偶极内场系数)
- 共振条件: $\mathrm{Re}(\varepsilon) = -2$(LSPR),此时 $\beta \to \infty$、 $k_\mathrm{in} \to 0$(理想金属屏蔽)

**电势表达式**(代码中 `laser_potential` 返回 $V^\mathrm{code} = -\Phi$,调用端加负号):

球外($r \geq a$): 
$$V^\mathrm{code}_\mathrm{out} = \mathrm{Re}[\mathrm{amp}] \cdot (\mathbf{pol} \cdot \Delta\mathbf{r}) - \mathrm{Re}[\beta \cdot \mathrm{amp}] \cdot a^3 \cdot \frac{\mathbf{pol} \cdot \Delta\mathbf{r}}{r^3}$$

球内($r < a$): 
$$V^\mathrm{code}_\mathrm{in} = \mathrm{Re}[k_in \cdot \mathrm{amp}] \cdot (\mathbf{pol} \cdot \Delta \mathbf{r})$$

其中 $\Delta\mathbf{r} = \mathbf{r} - \mathbf{r}_0$， $r = |\Delta\mathbf{r}|$， $\mathrm{amp} = f(t) e^{i(\omega t + \phi)}$ 为复包络。

**关键实现要点**:
1. 散射项为**负号**(源自 $V^\mathrm{code} = -\Phi_\mathrm{sca}$, $\Phi_\mathrm{sca} = +\beta a^3 E \cos\theta / r^2$)。
2. $\beta \cdot \mathrm{amp}$ 必须**复数整体相乘后取实部**,不能拆为 $\mathrm{Re}(\beta)\mathrm{Re}(\mathrm{amp})$,否则丢失相位信息。
3. **Maxwell 边界条件验证**(球面 $r = a$ 处):
   - 球外系数 $1 - \mathrm{Re}(\beta) = \mathrm{Re}\left[\dfrac{3}{\varepsilon+2}\right] = \mathrm{Re}(k_\mathrm{in})$ = 球内系数
   - ⇒ (1) $V$ 连续;(2) 切向 $E$ 连续;(3) 法向 $D = \varepsilon E$ 连续
4. **共振极限**($\varepsilon \to -2$):必须提供 $\varepsilon_\mathrm{im} > 0$ 避免数值发散,代码在 `laser_init` 中显式检查 `|eps + 2| < epsilon` 并 fatal 退出。

**输入参数**(共 6 个,原子单位):
- `r0_x, r0_y, r0_z`:球心坐标(a₀)
- `sphere_a`:球半径(a₀),必须 > 0
- `eps_re`:介电函数实部
- `eps_im`:介电函数虚部(必须 ≥ 0)

**典型金属参数参考**(可见光区,LSPR 附近):

| 金属 | 光子能量 | $\varepsilon_\mathrm{re}$ | $\varepsilon_\mathrm{im}$ | $|\beta|$ | 表面增强 $|1-\beta|^2/|k_\mathrm{in}|^2$ |
|---|---|---|---|---|---|
| Ag | 3.0 eV | -2.6 | 0.2 | ≈5.7 | ≈12.4 |
| Au | 2.3 eV | -2.5 | 1.5 | ≈3.2 | ≈4 |

---

## 三、TDExternalFields 输入格式

非均匀电场使用 `inhomogeneous_electric_field` 作为 type 关键字，可直接指定type=5。前 7 列与均匀电场相同(共用电场基础解析),第 8 列起为模型字符串与参数。

### 通用前缀(7 列)

```
inhomogeneous_electric_field | nx | ny | nz | omega | envelope_function_name | phase
```

### 模型 1:Taylor

```
%TDExternalFields
  5 | 0 | 0 | 1 | 1.0*eV | "env" | "0" | "taylor" | r0x | r0y | r0z | Gx | Gy | Gz | Hx | Hy | Hz
%
```

### 模型 2:Exponential

```
%TDExternalFields
  5 | 0 | 0 | 1 | 1.0*eV | "env" | "0" | "exponential" | r0x | r0y | r0z | ax | ay | az
%
```

### 模型 3:Dipole(Mie 纳米粒子)

```
%TDExternalFields
  5 | 0 | 0 | 1 | 3.0*eV | "env" | "0" | "dipole" | r0x | r0y | r0z | a | eps_re | eps_im
%
```

> 说明:`nx, ny, nz` 为极化向量(可复数);`omega` 为载频(能量单位);`envelope_function_name` 必须在 `TDFunctions` 块中定义;`phase` 可省略(默认为 0)。

---

## 四、势能空间分布输出

在 inp 文件中加入以下选项即可输出激光势能(包括非均匀电场势能)的空间分布:

```
Output = external_td_potential
OutputInterval = 10
OutputFormat = plane_x + cube
```

- `OutputFormat`:可选 `plane_x/y/z`(平面切片,文件小,适合快速查看)或 `cube`(Cube 格式,适合 VESTA/ChimeraX 可视化)。
- `OutputInterval`:每隔多少步输出一次。
- 输出文件位于 `td.<iter>/...` 目录下,文件名含 `external_td_potential`。

---

## 五、编译部署

Octopus 8.4 使用 Autotools 构建系统。修改的 `.F90` 文件**无需修改 Makefile**,只需替换服务器上对应文件后重新编译即可:

```bash
# 上传 6 个修改文件到服务器对应位置后:
cd /path/to/octopus-8.4
make clean
make          # 或 make -j N 并行编译
make install
```

> 无需重新运行 `./configure`,源码级修改直接 `make` 即可。

---

## 六、物理正确性验证要点

1. **Taylor 模型**:系数严格按积分关系,$G$ 项为 $\frac{1}{2}$、$H$ 项为 $\frac{1}{6} = \frac{1}{2} \cdot \frac{1}{3}$,源自 $\int E\,dr$ 的泰勒展开积分。

2. **Exponential 模型**:$\alpha \to 0$ 时显式退化为均匀场,$(\exp(\alpha \Delta r) - 1)/\alpha \to \Delta r$。

3. **Dipole 模型**(最关键):
   - 球面 $r = a$ 处势能严格连续:$1 - \mathrm{Re}(\beta) = \mathrm{Re}(k_\mathrm{in})$
   - 共振极限 $\varepsilon \to -2$ 时 $\beta \to \infty$、$k_\mathrm{in} \to 0$,符合金属屏蔽效应
   - 要求 $\varepsilon_\mathrm{im} \geq 0$,否则 $\beta$ 虚部符号反转导致场指数发散
   - 适用范围:球半径 $a \lesssim 30$ nm(准静态 / 偶极近似有效)

4. **单位一致性**:所有长度参数使用原子单位 $a_0$;介电函数无量纲;极化向量为电场幅度(原子单位)。

---

## 七、典型应用示例

### 示例:Ag 纳米球(半径 1.5 nm)近场增强计算

```
%TDFunctions
  "env" | tdf_from_expr | "1.0*exp(-(t-10*fs)^2/(5*fs)^2)"
%

%TDExternalFields
  5 | 0 | 0 | 1 | 3.0*eV | "env" | "0" | "dipole" | 0 | 0 | 0 | 5*nm | -2.6 | 0.2
%

Output = external_td_potential
OutputInterval = 20
OutputFormat = plane_x
```

说明:球心位于原点,半径 $a = 5$ nm,沿 z 方向极化,光子能量 3.0 eV(接近 Ag LSPR)。

---

## 八、修改文件对照表(快速定位)

| 功能点 | 文件位置 |
|---|---|
| 新电场类型常量定义 | `lasers.F90` L66-79 |
| `laser_t` 结构体扩展 | `lasers.F90` L92-107 |
| 输入参数解析 | `lasers.F90` L461-514 |
| 三种模型势能计算 | `lasers.F90` L753-832 |
| 模型信息输出 | `lasers.F90` L636-655 |
| 标量势路径合并 | `lasers_inc.F90` L116 |
| Hamiltonian 调用 | `hamiltonian.F90` L806, L1381 |
| 输出选项定义 | `output.F90` L280 |
| 势能输出实现 | `output_h_inc.F90` L307-333 |
| TD 输出调用 | `td_write.F90` L877-879 |

---

*文档生成日期:2026-07-30*
*基于 Octopus 8.4 源码修改*
