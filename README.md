# Triangulation by constraint Delaunay or Ear Clipping

> 使用约束德劳内或者耳切进行三角剖分，返回三角化后的geojson和mesh

![面数据三角化](https://github.com/bigsu/delaunay_earcut/assets/18549401/9a691fca-c45b-4023-bfd1-fb2ed9a6a2bc)


```
const fs = require('fs');
const path = require('path');
const turf = require('@turf/turf');
const terrain_decode = require("./11.22terrain_decode_sqlite")
const earcut = require('earcut');
const poly2tri = require('poly2tri');

const basePath = "D:/work/mapdata/output/pg_test/de"

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
    let filepath = path.join(basePath, "de.geojson");
    filepath = "D:/work/mapdata/output/pg_test/1699429536/14-26969-4552/road-14-26969-4552.geojson";

    let geojson = JSON.parse(fs.readFileSync(filepath));

    geojson = await terrain_decode.setGeojsonHeight(geojson);

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

    fs.writeFileSync(path.join(basePath, 'triangles.geojson'), JSON.stringify(geojson_triangles));
    fs.writeFileSync(path.join(basePath, 'triangles_mesh.json'), JSON.stringify(mesh));

    console.log('over');
    console.timeEnd("geojson")
}

getTrianglesMesh(true)
```

