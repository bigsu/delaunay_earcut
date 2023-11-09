# Triangulation by constraint Delaunay or Ear Clipping

> 使用约束德劳内或者耳切进行三角剖分，返回三角化后的geojson和mesh
> 
> https://github.com/r3mi/poly2tri.js
> 
> https://github.com/Turfjs/turf/issues/241

![面数据三角化](https://github.com/bigsu/delaunay_earcut/assets/18549401/9a691fca-c45b-4023-bfd1-fb2ed9a6a2bc)


```
const fs = require('fs');
const turf = require('@turf/turf');
const earcut = require('earcut');
const poly2tri = require('poly2tri');

/**
*   turf.constrainedtin
*   Function that gets a {@link Polygon} and make a tin constrained to it using Poly2tri or Earcut
* @requires [poly2tri.js]{@link http://r3mi.github.io/poly2tri.js/dist/poly2tri.js}
* @requires  [earcut.js]{@link https://github.com/mapbox/earcut}
* @param {Feature<(Polygon)>} poly - single Polygon Feature
* @param {Boolean} delaunay - retrieves a Delaunay triangulation
* @return {FeatureCollection<(Polygon)>} 
* @author   Abel Vázquez
* @version 1.0.0
*/
function constrainedtin(poly, delaunay) {
    if (poly.geometry === void 0 || poly.geometry.type !== 'Polygon') throw ('"polygonreduce" only accepts polygon type input');
    delaunay = (delaunay !== void 0) ? delaunay : false;
    let features = [];
    let mesh = { points: [], triangles: [] };
    if (delaunay === true) {
        const allCoordinates = [];
        var
            ctx = poly.geometry.coordinates.map(function (a) {
                var c = a.map(function (b) {
                    allCoordinates.push(b);
                    // return new poly2tri.Point(b[0], b[1], b[2])
                    return { x: b[0], y: b[1], z: b[2] }
                });
                c.pop();
                return c;
            }),
            swctx = new poly2tri.SweepContext(ctx.shift());
        // 获取去重后的点数组
        const vertices = allCoordinates.filter((coord, index, self) => {
            return self.findIndex(c => c[0] === coord[0] && c[1] === coord[1]) === index;
        });
        if (ctx.length > 0) swctx.addHoles(ctx);
        features = swctx.triangulate().getTriangles().map(function (t) {
            var points = [];
            t.getPoints().forEach(function (p) {
                points.push([p.x, p.y, p.z]);
            });
            points.push([t.getPoint(0).x, t.getPoint(0).y, t.getPoint(0).z])
            //3个点
            const tri = points.slice(0, 3).map(coord => {
                return vertices.findIndex(p => p[0] === coord[0] && p[1] === coord[1]);
            });
            mesh.triangles.push(tri);
            return turf.polygon([points]);
        });
        mesh.points = vertices;
    } else {
        const vertices = poly.geometry.coordinates;
        let points = vertices.flat(1);
        const data = earcut.flatten(vertices);
        var
            t = earcut(data.vertices, data.holes, data.dimensions),
            p, tri;
        for (var i = 0; i < t.length - 2; i += 3) {
            p = [points[t[i]], points[t[i + 1]], points[t[i + 2]], points[t[i]]];
            tri = [t[i], t[i + 1], t[i + 2]];
            features.push(turf.polygon([p]));
            mesh.triangles.push(tri);
        }
        mesh.points = points;
    }
    return { f: turf.featureCollection(features), m: mesh };
}

async function getTrianglesMesh(toDelaunay) {
    console.time("geojson")

    let geojson = { "type": "FeatureCollection", "features": [{ "type": "Feature", "properties": {}, "geometry": { "coordinates": [[[116.38873371939997, 39.91738262404331], [116.39014334177779, 39.91501371845206], [116.39249570523612, 39.91516522068099], [116.393752693343, 39.91818830872441], [116.39140603325404, 39.917085497984544], [116.38873371939997, 39.91738262404331]], [[116.39180300058342, 39.91663216133148], [116.39234001685924, 39.916033078006876], [116.39149688558405, 39.916327526308464], [116.39105016891176, 39.91570924864908], [116.39099501033331, 39.91649121780645], [116.39180300058342, 39.91663216133148]], [[116.3898495136317, 39.91694339996246], [116.39050766634921, 39.91694339996246], [116.39050766634921, 39.91642331444925], [116.3898495136317, 39.91642331444925], [116.3898495136317, 39.91694339996246]]], "type": "Polygon" } }, { "type": "Feature", "properties": {}, "geometry": { "coordinates": [[[[116.38975976553297, 39.91880955664365], [116.39039797422987, 39.918312429709744], [116.38948054922804, 39.91793002192048], [116.39066721852299, 39.917547611995786], [116.39228268428496, 39.9181594668504], [116.39171427966471, 39.919230199691896], [116.38975976553297, 39.91880955664365]]], [[[116.39293086499055, 39.919398456187196], [116.3923724323821, 39.918824852799446], [116.39329982939267, 39.91834302224058], [116.39370868183903, 39.91905429473124], [116.39293086499055, 39.919398456187196]]]], "type": "MultiPolygon" } }] }

    let mesh = [];
    const geojson_triangles = {
        type: 'FeatureCollection',
        features: []
    };
    await Promise.all(geojson.features.map(async (feature) => {
        if (feature.geometry.type === 'Polygon') {
            const { f, m } = constrainedtin(feature, toDelaunay);
            geojson_triangles.features.push(...f.features);
            mesh.push(m);
        } else if (feature.geometry.type === 'MultiPolygon') {
            var flatten = turf.flatten(feature);
            flatten.features.map(async (polygon) => {
                const { f, m } = constrainedtin(polygon, toDelaunay);
                geojson_triangles.features.push(...f.features);
                mesh.push(m);
            });
        }
    }));

    fs.writeFileSync('triangles.geojson', JSON.stringify(geojson_triangles));
    fs.writeFileSync('triangles_mesh.json', JSON.stringify(mesh));

    console.log('over');
    console.timeEnd("geojson")
}

getTrianglesMesh(true)
```

