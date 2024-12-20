# 📝 Misc Guidelines

(I will keep updating this)
- Use genesis.tensor whenever possible. Note that when we pass genesis tensor to taichi kernels, call tensor.assert_contiguous() to check whether it's contiguous, since taichi only support contiguous external tensor.
- Don't expose any taichi-related usage to user.
- When you add new class, also implement `__repr__()` for easier interactive debugging. (See genesis/engine/states.py as an example)
- Contain all simulation-related stuff in genesis.engine.
- Use `genesis.logger.info()`/`debug()`/`warning()` instead of `print()`.
- Users will be alerted that it's not recommended to query for scene states too many times and not use them. All accessed scene states will be stored in a scene-level list, and considered part of the computation graph. This list will be freed when calling `scene.reset_grad()`, which frees up all occupied gpu memory.
- Hierarchy - we abstract each level of entity creation, so they are unified and independent of each other:
    - physics solver: we support various types: MPM, PBD, SPH, FEM, rigid, etc. The idea is to let user flexibly choose among them, without having to change any front-end APIs
    - material -> this determined backend physics solver. we will have MPMLiquid, SPHLiquid, PBDLiquid, MPMElastic, FEMElastic, etc.
    - geom -> this defines the entity's geometry. Can be either one of the shape primitives, or from mesh, or from URDF, etc. These geometries are independent of solver used.
    - all different entities are added via the same `scene.add_entity()`
- default solver order (for code consistency)
    - rigid
    - avatar
    - mpm
    - sph
    - pbd
    - fem
    - sf
- Some order convention
    - quaternion: `[w, x, y, z]`
    - euler
        - user input: we use extrinsic x-y-z, in `degree`, as this is more intuitive
            - interpretation: we use `scipy.Rotation`'s `xyz` ordering.
        - internal xyz:
            - euler is defined differently in various sources
            - in our case, we use xyz to refer to the intrinsic rotation in order x-y-z, same as mujoco. Note that this aligns with others, e.f. dof force and position.
            - for angular velocity, we use rotvec.
- We use `id` for each object's `uuid`, and `idx` for its index
- `uv` order
    - assimp, trimesh: from bottom left corner
    - ours, pygltflib, luisa: from top left corner
- sim options vs solver options
    - for any parameter that exists both in sim and solver options, the one in solver options has a higher priority, and will be initialized using the value in sim options when undefined
    - the recommended way is to define dt via sim options, so that all solvers operate in the same temporal scale. However, users can also set different dt for different solvers
    - RigidSolver operates at step level, and all other solvers operate at substep level. In order to make them compatible, all non-rigid solvers use `substeps` in sim options.
- Some design and conventions for rigid solver
    - for attribute, we use `*_idx`. e.g. `link_info.parent_idx`
    - for id inside loop iteration, we use `i_*`.
    - For all variables inside kernel loop
        - suffix abbreviation:
            - `i_l`: link id
            - `i_p`: parent link id
            - `i_r`: root link id
            - `i_g`: geom id
            - `i_d`: dof
        - for prefix, we use:
            - `l_`: link
            - `l_info`: links_info[i_l, i_b]
            - `g_`: geom
            - `p_`: parent
            - ...
    - on index storing
        - Do we store offset-ed index in each class (`link`, `geom`, etc) or only a local one?
            - let's do the former, since when user query e.g., an `entity`, it's better if it shows the global link idx
            - This applies to indexes of `link`, `geom`, `verts`, etc.
        - each object stores
            - its parent class. e.g. `link` stores its `entity`
            - its global `idx` after offset
            - offset value for its own children. e.g. a `geom` only stores `idx_offset_*` for `vert`, `face`, and `edge`.
    - root vs base
        - `root` is a more natural name as we use a tree structure for the links, while `base` is more informative from user's perspective
        - Let's use `root` link inernally, and `base` link for docs etc.
    - root pose vs q (current design subject to possible change)
        - for both arm and single mesh, its pos and euler specified when loaded is its root pose, and q will be w.r.t to this
    - control interface
        - root pos should be merged with joint pos of the first (fixed) joint, which connectes world to the first link
        - if we want to control something, we will command velocity
            - thus, if it’s a free joint, it’s ok. We will override velocity
            - if it’s a fixed joint, there’s no dof for it, so we cannot control it
        - if we need position control, we will write a PD controller and send velocity command under the hood.
        - we can still change root pos (first joint pos), even if its fixed. but this is NOT recommended.
            - this applies to both fixed and free joint. Not recommended in both case, because even it’s free joint, setting position will violate physics.
            - then what’s the difference between free and fixed joint?
                - the free dof will be affected by external effect, while the fixed joint won’t
    - mjcf vs urdf
        - mjcf xml always has a worldbody, so we will skip it. sometimes this worldbody has some geoms associated, let’s not support it for now.
        - urdf only has robot, so we will load everything. Sometimes the robot can have a included world link, then it will be loaded into genesis's world and be the root link of the entity.
    - collision handling: we store base on convexelized geoms
        - for mesh-based assets, we generate a convex hull of all the groups stored in the mesh
            - this groups can either be originally stored submeshes, or if group_by_material=True, we will group by material
            - each group will be one RigidGeom
        - for mjcf, we convexlize based on mujoco's geoms. Each mj geom will be one RigidGeom
        - for urdf
            - each urdf can contain multiple links, each link contains multiple geometries (collisions and visuals), and each geomtry will be one primitive or one external assset. Since `.obj` contains multiple sub-meshes, one urdf geomtry can have multiple meshes
            - we convexlize this lowest-level mesh and store as RigidGeom
    - control interface design
        - we will not explicitly have concept like `base pose`
            - in pybullet, movable mesh is its own baselink, and when pushed its base pose will change
            - in genesis
                - everything will connect to world (link -1)
                - everything will have root pose. This is the initial pose and will not be changed. This is the reference we use when calculating q.
                - free moving objects will connect to world via a free joint with 6 DoFs. When being pushed, this state will be changed, but its root pose will stay the same.
    - prefix `v`
        - this is used for global parameters that are used for visualization (visual geoms, verts, edges, normals etc)
- `surface.vis_mode`:
    - For rigid bodies, supported modes are ['visual', 'collision', 'sdf']. Default type is `visual`.
    - For deformable non-fluid bodies, supported modes are ['visual', 'particle', 'recon']. Default type is `visual`.
        - `visual` will render the input full visual mesh, skinned using internal particle state
        - `particle` will render the internal particles. If the input texture is a color texture, the color will be used. In case of a image texture, the particles will be rendered using the texture's mean_color.
        - `recon` will perform surface reconstruction using the particles.
    - For fluid bodies, supported modes are ['particle', 'recon']. Default type is `particle`.
        - `particle` will render the internal particles. If the input texture is a color texture, the color will be used. In case of a image texture, the particles will be rendered using the texture's mean_color.
        - `recon` will perform surface reconstruction using the particles.
    '''